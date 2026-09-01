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
    private readonly HttpClient _http;

    public LocalBrain(OctaviaConfig config)
    {
        _config = config;
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
        history.Add(Utterance.User, userText);

        var messages = new List<object> { new { role = "system", content = Persona.System } };
        messages.AddRange(history.Turns.Select((t, i) => new
        {
            role = t.Role,
            // now.Image is ignored: this brain has no eyes. See IBrain.
            content = Persona.Situated(t.Text, now.Context, i == history.Turns.Count - 1)
        }));

        var request = new
        {
            model = _config.LocalModel,
            messages,
            stream = true,
            max_tokens = _config.MaxTokens,
            temperature = 0.7
        };

        HttpResponseMessage response;
        try
        {
            response = await _http.PostAsJsonAsync("chat/completions", request, cancel);
        }
        catch (OperationCanceledException) when (cancel.IsCancellationRequested)
        {
            history.DropLast();
            throw;
        }
        catch (Exception ex)
        {
            history.DropLast();
            Log.Write($"local brain unreachable: {ex.Message}");
            throw new InvalidOperationException(
                $"No local model server at {_config.LocalEndpoint}. Start Ollama or LM Studio.");
        }

        using (response)
        {
            if (!response.IsSuccessStatusCode)
            {
                var detail = await response.Content.ReadAsStringAsync(cancel);
                history.DropLast();
                Log.Write($"local brain {(int)response.StatusCode}: {detail}");
                throw new InvalidOperationException(
                    $"Local model refused the request ({(int)response.StatusCode}). " +
                    $"Is '{_config.LocalModel}' pulled?");
            }

            var spoken = new StringBuilder();
            var pending = new StringBuilder();
            var think = new Speech.ThinkFilter();

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

                var chunk = ContentOf(payload);
                if (chunk.Length == 0) continue;

                // Small reasoning models narrate their scratchpad; never say it aloud.
                var visible = think.Filter(chunk);
                if (visible.Length == 0) continue;

                spoken.Append(visible);
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

            var tail = Speech.Speakable(pending.ToString());
            if (tail.Length > 0) yield return tail;

            if (spoken.Length == 0)
            {
                history.DropLast();
                throw new InvalidOperationException("The local model returned nothing.");
            }

            history.Add(Utterance.Assistant, Speech.Speakable(spoken.ToString()));
        }
    }

    private static string ContentOf(string payload)
    {
        try
        {
            using var doc = JsonDocument.Parse(payload);
            if (!doc.RootElement.TryGetProperty("choices", out var choices) ||
                choices.GetArrayLength() == 0) return string.Empty;

            var choice = choices[0];
            if (choice.TryGetProperty("delta", out var delta) &&
                delta.TryGetProperty("content", out var content) &&
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
