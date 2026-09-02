---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\LocalBrain.cs
---

# src\Octavia.Core\Brain\LocalBrain.cs

```csharp
using System.Net.Http.Json;
using System.Runtime.CompilerServices;
using System.Text;
using System.Text.Json;
using Octavia.Core;

namespace Octavia.Brain;

/// Talks to any server exposing the OpenAI-compatible chat API — Ollama, LM Studio,
/// llama-server. Deliberately out-of-process: a second CUDA-linked native runtime in
/// this process would collide with Whisper, and a server can unload its model when
/// the GPU is wanted elsewhere.
internal sealed class LocalBrain : IBrain
{
    private readonly OctaviaConfig _config;
    private readonly Tools.ToolRegistry? _tools;
    private readonly HttpClient _http;

    /// The same ceiling `ClaudeBrain` uses, for the same reason: a model that answers its
    /// own tool result with another call has no natural stopping point. It costs nothing
    /// here but time — which on a CPU-pinned brain is the expensive part.
    private const int MaxToolRounds = 4;

    public LocalBrain(OctaviaConfig config, Tools.ToolRegistry? tools = null)
    {
        _config = config;
        _tools = tools;
        _http = new HttpClient
        {
            BaseAddress = new Uri(config.LocalEndpoint.TrimEnd('/') + "/"),
            Timeout = TimeSpan.FromMinutes(5)
        };
    }

    public string Description => $"{_config.LocalModel} (local)";
    public bool IsReady => true;          // failure surfaces on use, with a usable message
    public bool NeedsApiKey => false;

    public async IAsyncEnumerable<string> RespondAsync(
        Conversation history,
        string userText,
        Situation now = default,
        [EnumeratorCancellation] CancellationToken cancel = default)
    {
        // Taken before the turn begins, and cleared by the taking.
        var consent = history.TakeConsent();

        history.Add(Utterance.User, userText);

        var messages = new List<object> { new { role = "system", content = Persona.System } };
        messages.AddRange(history.Turns.Select((t, i) => new
        {
            role = t.Role,
            // now.Image is ignored: this brain has no eyes. See IBrain.
            content = Persona.Situated(t.Text, now.Context, i == history.Turns.Count - 1)
        }));

        var spoken = new StringBuilder();

        /* Asked once per turn rather than held: a server can come up or fall over between
           one question and the next, and the registry is where that is known.

           **The request is built identically when there is nothing to offer** — no `tools`
           key, nothing reordered. That matters more here than it does for the hosted brain:
           a model with no tool template will refuse a `tools` array outright on some
           servers, so a machine with no tool servers configured must not start sending one. */
        var offered = _tools is null
            ? []
            : await _tools.ListAsync(cancel);

        for (var round = 0; ; round++)
        {
            var calls = new Dictionary<int, Wanted>();
            var pending = new StringBuilder();
            var said = new StringBuilder();
            var think = new Speech.ThinkFilter();

            HttpResponseMessage response;
            try
            {
                response = await _http.PostAsJsonAsync("chat/completions", Ask(messages, offered), cancel);
            }
            catch (OperationCanceledException) when (cancel.IsCancellationRequested)
            {
                // Only the turn that has said nothing yet is safe to withdraw. Dropping it
                // after she has spoken would leave the history claiming she answered a
                // question nobody asked.
                if (spoken.Length == 0) history.DropLast();
                throw;
            }
            catch (Exception ex)
            {
                if (spoken.Length == 0) history.DropLast();
                Log.Write($"local brain unreachable: {ex.Message}");
                throw new InvalidOperationException(
                    $"No local model server at {_config.LocalEndpoint}. Start Ollama or LM Studio.");
            }

            using (response)
            {
                if (!response.IsSuccessStatusCode)
                {
                    var detail = await response.Content.ReadAsStringAsync(cancel);
                    if (spoken.Length == 0) history.DropLast();
                    Log.Write($"local brain {(int)response.StatusCode}: {detail}");
                    throw new InvalidOperationException(
                        $"Local model refused the request ({(int)response.StatusCode}). " +
                        $"Is '{_config.LocalModel}' pulled?");
                }

                await using var body = await response.Content.ReadAsStreamAsync(cancel);
                using var reader = new StreamReader(body);

                // Read to null rather than testing EndOfStream: that property has to peek at
                // the stream to answer, and on a server-sent-event response there is nothing
                // to peek at until the model emits the next token. It blocked a thread pool
                // thread for the length of every reply — worst on the CPU-pinned brain, which
                // is the one that is slowest to produce a token.
                while (await reader.ReadLineAsync(cancel) is { } line)
                {
                    if (cancel.IsCancellationRequested) break;

                    if (string.IsNullOrWhiteSpace(line) || !line.StartsWith("data:")) continue;

                    var payload = line[5..].Trim();
                    if (payload == "[DONE]") break;

                    var chunk = Absorb(payload, calls);
                    if (chunk.Length == 0) continue;

                    // Small reasoning models narrate their scratchpad; never say it aloud.
                    var visible = think.Filter(chunk);
                    if (visible.Length == 0) continue;

                    spoken.Append(visible);
                    said.Append(visible);
                    pending.Append(visible);

                    foreach (var sentence in Speech.DrainSentences(pending))
                    {
                        var speakable = Speech.Speakable(sentence);
                        if (speakable.Length > 0) yield return speakable;
                    }
                }

                // Whatever the filter was still holding is real output too — count it,
                // or a reply short enough to be held whole looks like an empty response.
                var flushed = think.Flush();
                pending.Append(flushed);
                spoken.Append(flushed);
                said.Append(flushed);

                var tail = Speech.Speakable(pending.ToString());
                if (tail.Length > 0) yield return tail;
            }

            var wanted = calls.OrderBy(pair => pair.Key).Select(pair => pair.Value).ToList();

            if (wanted.Count == 0 || _tools is null) break;

            if (round >= MaxToolRounds)
            {
                Log.Warn($"stopping after {MaxToolRounds} rounds of tools; she was still asking");
                break;
            }

            /* The assistant turn carries the calls back verbatim and every one of them is
               answered in the messages that follow. A `tool` message whose id matches
               nothing — or a call left unanswered — is rejected outright by servers that
               implement this properly and, worse, quietly ignored by some that do not. */
            var reached = new List<object>();
            var answers = new List<object>();

            foreach (var call in wanted)
            {
                var name = call.Name.ToString();
                var arguments = call.Arguments.ToString().Trim();
                if (arguments.Length == 0) arguments = "{}";

                reached.Add(new
                {
                    id = call.Id.ToString(),
                    type = "function",
                    function = new { name, arguments }
                });

                // The same rule as `ClaudeBrain`, asked of the same place: a yes spoken in
                // the previous turn, for this exact call, and nothing older.
                var granted = Conversation.Grants(consent, name, arguments, userText);

                if (granted) Log.Write($"tool '{name}': confirmed by the last thing said");

                var answer = await Answer(name, arguments, granted, cancel);

                if (!granted && answer.StartsWith("Not done:", StringComparison.Ordinal))
                    history.AwaitYes(name, arguments);

                answers.Add(new { role = "tool", tool_call_id = call.Id.ToString(), content = answer });
            }

            messages.Add(new { role = "assistant", content = said.ToString(), tool_calls = reached });
            messages.AddRange(answers);
        }

        if (spoken.Length == 0)
        {
            history.DropLast();
            throw new InvalidOperationException("The local model returned nothing.");
        }

        history.Add(Utterance.Assistant, Speech.Speakable(spoken.ToString()));
    }

    /// One tool call being assembled.
    ///
    /// **Every field accumulates, the id and the name included.** Ollama sends a call whole
    /// in a single chunk; the OpenAI streaming shape allows the arguments to arrive as
    /// fragments across many, and nothing forbids the rest from doing the same. Appending
    /// costs nothing when there is only ever one fragment and is correct when there is not.
    private sealed record Wanted(StringBuilder Id, StringBuilder Name, StringBuilder Arguments);

    /// The request, built the same way every round so the only thing that changes between
    /// laps is the messages.
    private object Ask(List<object> messages, IReadOnlyList<Tools.Tool> offered)
    {
        // Two shapes rather than one with a conditional key, so "identical when there is
        // nothing to offer" is something a reader can check rather than trust.
        if (offered.Count == 0)
            return new
            {
                model = _config.LocalModel,
                messages,
                stream = true,
                max_tokens = _config.MaxTokens,
                temperature = 0.7
            };

        return new
        {
            model = _config.LocalModel,
            messages,
            stream = true,
            max_tokens = _config.MaxTokens,
            temperature = 0.7,
            tools = offered.Select(Describe).ToList()
        };
    }

    /// One of hers in the shape this API wants, which nests what Anthropic keeps flat.
    ///
    /// The schema is passed through rather than rewritten — it came from the server that
    /// declared it, and a client that reshapes somebody else's schema is one that will
    /// eventually disagree with that server about what its own tool accepts.
    private static object Describe(Tools.Tool tool) => new
    {
        type = "function",
        function = new
        {
            name = tool.Name,
            description = tool.Description,
            parameters = tool.Schema
        }
    };

    /// Runs one call and returns what should go back as the `tool` message.
    ///
    /// Failures come back as text rather than as exceptions, because that is what the seam
    /// promised and because a model handed *"the gateway could not be reached"* can say so,
    /// where a thrown exception ends the turn with nothing to relay.
    private async Task<string> Answer(string name, string arguments, bool confirmed, CancellationToken cancel)
    {
        JsonElement parsed;

        try
        {
            parsed = JsonDocument.Parse(arguments).RootElement.Clone();
        }
        catch (JsonException ex)
        {
            Log.Warn($"a tool call's arguments would not parse ({ex.Message}); treating it as empty");
            parsed = JsonDocument.Parse("{}").RootElement.Clone();
        }

        var answer = await _tools!.CallAsync(name, parsed, confirmed, cancel);

        /* The text only. **This brain has no eyes** — the same sentence `IBrain` already
           uses about `Situation.Image` — so a picture would be bytes it cannot read and a
           great many tokens it would pay for. That is why a tool answering with an image is
           required to say in words what it captured: this path gets the words, and a turn
           here is still a usable one rather than a blank stare. */
        if (answer.HasImage)
            Log.Write($"tool '{name}' returned an image; {_config.LocalModel} has no eyes, using its words");

        return answer.Text;
    }

    /// Whatever text a chunk carried, with any tool-call fragments folded into `calls`.
    private static string Absorb(string payload, Dictionary<int, Wanted> calls)
    {
        try
        {
            using var doc = JsonDocument.Parse(payload);
            if (!doc.RootElement.TryGetProperty("choices", out var choices) ||
                choices.GetArrayLength() == 0) return string.Empty;

            var choice = choices[0];
            if (!choice.TryGetProperty("delta", out var delta)) return string.Empty;

            if (delta.TryGetProperty("tool_calls", out var asked) &&
                asked.ValueKind == JsonValueKind.Array)
            {
                foreach (var call in asked.EnumerateArray())
                {
                    // The index is what identifies a call across chunks, not the id — the
                    // id may only ever appear in the first fragment.
                    var index = call.TryGetProperty("index", out var at) && at.TryGetInt32(out var n)
                        ? n
                        : 0;

                    if (!calls.TryGetValue(index, out var building))
                        calls[index] = building = new Wanted(new(), new(), new());

                    if (call.TryGetProperty("id", out var id) && id.ValueKind == JsonValueKind.String)
                        building.Id.Append(id.GetString());

                    if (call.TryGetProperty("function", out var function))
                    {
                        if (function.TryGetProperty("name", out var name) &&
                            name.ValueKind == JsonValueKind.String)
                            building.Name.Append(name.GetString());

                        if (function.TryGetProperty("arguments", out var arguments) &&
                            arguments.ValueKind == JsonValueKind.String)
                            building.Arguments.Append(arguments.GetString());
                    }
                }
            }

            if (delta.TryGetProperty("content", out var content) &&
                content.ValueKind == JsonValueKind.String)
                return content.GetString() ?? string.Empty;

            return string.Empty;
        }
        catch (JsonException)
        {
            return string.Empty;
        }
    }

    public void Dispose() => _http.Dispose();
}
```
