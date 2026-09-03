---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\ClaudeBrain.cs
---

# src\Octavia.Core\Brain\ClaudeBrain.cs

```csharp
using System.Runtime.CompilerServices;
using System.Text;
using System.Text.Json;
using System.Text.Json.Nodes;
using Anthropic;
using Anthropic.Models.Messages;
using Octavia.Brain.Tools;
using Octavia.Core;

// Both namespaces call their tool a `Tool`, and both are needed in this one file.
using WireTool = Anthropic.Models.Messages.Tool;

namespace Octavia.Brain;

internal sealed class ClaudeBrain : IBrain
{
    private readonly OctaviaConfig _config;
    private readonly ToolRegistry? _tools;
    private AnthropicClient? _client;
    private string? _keyInUse;

    /// How many times she may call tools and think again inside one turn.
    ///
    /// A ceiling rather than a target. The failure it exists for is a model that answers
    /// its own tool result with another tool call forever — which costs money on every
    /// lap and, unlike a slow answer, never ends on its own. Four is enough for *look
    /// something up, then look up what that pointed at*, which is the deepest chain any
    /// of the read-only tools can currently justify.
    private const int MaxToolRounds = 4;

    public ClaudeBrain(OctaviaConfig config, ToolRegistry? tools = null)
    {
        _config = config;
        _tools = tools;
    }

    public string Description => _config.Model;
    public bool IsReady => !string.IsNullOrWhiteSpace(SecretStore.ReadApiKey());
    public bool NeedsApiKey => true;

    private AnthropicClient Client()
    {
        var key = SecretStore.ReadApiKey()
                  ?? throw new InvalidOperationException("No Anthropic API key has been set.");

        if (_client is null || _keyInUse != key)
        {
            _client = new AnthropicClient { ApiKey = key };
            _keyInUse = key;
        }

        return _client;
    }

    /// One turn of the history as the API wants it. Only the newest carries the
    /// situation: a photograph taken a quarter of an hour ago is not what she can see
    /// now, and leaving it in the history would have her describing it forever.
    private static MessageParam Turn(Utterance turn, Situation now, bool isCurrent)
    {
        var role = turn.Role == Utterance.User ? Role.User : Role.Assistant;
        var text = Persona.Situated(turn.Text, now.Context, isCurrent);

        if (!isCurrent || now.Image is null)
            return new MessageParam { Role = role, Content = text };

        // The image goes *before* the words. The question is almost always about the
        // picture, and a model reads the blocks in the order they are given.
        return new MessageParam
        {
            Role = role,
            Content = new List<ContentBlockParam>
            {
                new ImageBlockParam
                {
                    Source = new Base64ImageSource { Data = now.Image, MediaType = "image/jpeg" }
                },
                new TextBlockParam { Text = text }
            }
        };
    }

    /// One tool call, assembled from the stream.
    ///
    /// The arguments arrive as `input_json_delta` fragments that mean nothing until the
    /// block ends, so the JSON is accumulated rather than parsed as it lands.
    private sealed record Call(string Id, string Name, StringBuilder Json);

    public async IAsyncEnumerable<string> RespondAsync(
        Conversation history,
        string userText,
        Situation now = default,
        [EnumeratorCancellation] CancellationToken cancel = default)
    {
        // Resolve the client first: a missing key must not leave a dangling user turn
        // in the history, or the next request sends two user messages in a row.
        var client = Client();

        /* Taken before the turn begins, and cleared by the taking. Whatever she asked about
           last time is answerable by this utterance and by nothing after it. */
        var consent = history.TakeConsent();

        history.Add(Utterance.User, userText);

        /* Asked once per turn rather than held, because a server can come up or fall over
           between one question and the next and the registry is where that is known.

           **The request is built identically when there is nothing to offer.** No `Tools`,
           no extra block, no reordering — a machine with no servers configured sends the
           byte-for-byte request it sent before this existed, which is what keeps the
           system-prompt cache breakpoint working and this change off the critical path for
           everybody who has no house attached. */
        var offered = _tools is null
            ? []
            : await _tools.ListAsync(cancel);

        var messages = history.Turns
            .Select((t, i) => Turn(t, now, i == history.Turns.Count - 1))
            .ToList();

        var parameters = Ask(messages, offered);

        var spoken = new StringBuilder();
        var pending = new StringBuilder();
        var failed = false;

        for (var round = 0; ; round++)
        {
            var said = new StringBuilder();
            var building = new Dictionary<long, Call>();

            var stream = client.Messages.CreateStreaming(parameters, cancellationToken: cancel);
            var enumerator = stream.GetAsyncEnumerator(cancel);

            try
            {
                while (true)
                {
                    string? chunk = null;
                    try
                    {
                        if (!await enumerator.MoveNextAsync()) break;
                        var moment = enumerator.Current;

                        if (moment.TryPickContentBlockStart(out var opened) &&
                            opened.ContentBlock.TryPickToolUse(out var wants))
                        {
                            building[opened.Index] = new Call(wants.ID, wants.Name, new StringBuilder());
                        }
                        else if (moment.TryPickContentBlockDelta(out var delta))
                        {
                            if (delta.Delta.TryPickText(out var text))
                            {
                                chunk = text.Text;
                            }
                            else if (delta.Delta.TryPickInputJson(out var arguments) &&
                                     building.TryGetValue(delta.Index, out var call))
                            {
                                call.Json.Append(arguments.PartialJson);
                            }
                        }
                    }
                    catch (OperationCanceledException)
                    {
                        throw;
                    }
                    catch (Exception ex)
                    {
                        Log.Error("claude stream failed", ex);
                        failed = true;
                        break;
                    }

                    if (chunk is null) continue;

                    spoken.Append(chunk);
                    said.Append(chunk);
                    pending.Append(chunk);

                    foreach (var sentence in Speech.DrainSentences(pending))
                        yield return sentence;
                }
            }
            finally
            {
                await enumerator.DisposeAsync();
            }

            var calls = building.OrderBy(pair => pair.Key).Select(pair => pair.Value).ToList();

            if (failed || calls.Count == 0 || _tools is null) break;

            if (round >= MaxToolRounds)
            {
                // Said out loud rather than swallowed: a turn that stops because it hit a
                // ceiling looks exactly like one that simply finished.
                Log.Warn($"stopping after {MaxToolRounds} rounds of tools; she was still asking");
                break;
            }

            /* Everything she said before reaching for a tool is spoken already, and has to
               go back to her as well — an assistant turn that dropped its own words would
               have her reading a tool result she has no memory of asking for. */
            var reached = new List<ContentBlockParam>();
            if (said.Length > 0) reached.Add(new TextBlockParam { Text = said.ToString() });

            var answers = new List<ContentBlockParam>();

            foreach (var call in calls)
            {
                var arguments = Arguments(call.Json);

                // The wire wants the arguments as fields; the registry wants them whole.
                // Same object, read twice, rather than parsed twice.
                reached.Add(new ToolUseBlockParam
                {
                    ID = call.Id,
                    Name = call.Name,
                    Input = Fields(arguments)
                });

                /* A yes spoken in the previous turn, and only the previous one.
                   `Conversation` holds the rule; both brains ask it the same question. */
                var raw = arguments.GetRawText();

                Log.Debug($"tool call: {call.Name}{raw}");
                var granted = Conversation.Grants(consent, call.Name, raw, userText);

                if (granted) Log.Write($"tool '{call.Name}': confirmed by the last thing said");

                var answer = await _tools.CallAsync(call.Name, arguments, granted, cancel);

                /* **What the tool actually said, next to what she then says about it.**

                   Without this line the log records that a tool was called and nothing about
                   how it went, so *"she said she power-cycled the port and the camera never
                   blinked"* could not be told apart from *"the gateway refused and she said
                   it anyway"* — which is the exact question this cost a session to answer.
                   Truncated, because a client list is a page and a camera is 20 KB. */
                Log.Debug($"tool answer: {call.Name} -> {Speech.Brief(answer.Text)}");

                // Refused for want of a yes, so remember what was asked about. She is about
                // to put the question; the answer has one turn to arrive.
                if (!granted && answer.Text.StartsWith("Not done:", StringComparison.Ordinal))
                    history.AwaitYes(call.Name, raw);

                /* One result per call, in the same message: the API rejects a follow-up
                   where any tool_use has no matching tool_result.

                   A tool that returned a picture puts it *inside* its own result, rather
                   than as a loose image somewhere in the conversation. That is what keeps
                   "this is what the camera saw when you asked" attached to the asking — a
                   frame floating in the history would be a photograph with no caption the
                   moment a second one arrived. The words go after the image for the reason
                   `Turn` gives: the question is about the picture, and a model reads the
                   blocks in the order it is given them. */
                answers.Add(answer.HasImage
                    ? new ToolResultBlockParam
                    {
                        ToolUseID = call.Id,
                        Content = new ToolResultBlockParamContent(new List<Block>
                        {
                            new ImageBlockParam
                            {
                                Source = new Base64ImageSource
                                {
                                    Data = answer.Image!,
                                    MediaType = answer.ImageMediaType ?? "image/jpeg"
                                }
                            },
                            new TextBlockParam { Text = answer.Text }
                        })
                    }
                    : new ToolResultBlockParam { ToolUseID = call.Id, Content = answer.Text });
            }

            messages.Add(new MessageParam { Role = Role.Assistant, Content = reached });
            messages.Add(new MessageParam { Role = Role.User, Content = answers });
            parameters = Ask(messages, offered);
        }

        var tail = pending.ToString().Trim();
        if (tail.Length > 0) yield return tail;

        if (failed && spoken.Length == 0)
        {
            history.DropLast();
            throw new InvalidOperationException("The model could not be reached.");
        }

        /* Only what she *said* is remembered, not the tool exchange that produced it.

           `Conversation` holds strings, and widening it to structured blocks would change
           what every brain and the diagnostics bundle handle. It costs less than it looks:
           whatever the tools told her is in the sentence she just spoke, so the next turn
           still has the substance — she simply cannot quote the raw result back. When that
           starts to matter, the fix is a real one and belongs on its own. */
        history.Add(Utterance.Assistant, spoken.ToString());
    }

    /// The request, built the same way every round so the only thing that changes between
    /// laps is the messages.
    private MessageCreateParams Ask(List<MessageParam> messages, IReadOnlyList<Tools.Tool> offered)
    {
        var system = new List<TextBlockParam>
        {
            new() { Text = Persona.System, CacheControl = new CacheControlEphemeral() }
        };

        // Two initializers rather than one and a null, so "identical when there is nothing
        // to offer" is something you can read rather than something you have to trust the
        // serialiser about.
        if (offered.Count == 0)
            return new MessageCreateParams
            {
                Model = _config.Model,
                MaxTokens = _config.MaxTokens,
                Thinking = new ThinkingConfigDisabled(),
                System = system,
                Messages = messages
            };

        return new MessageCreateParams
        {
            Model = _config.Model,
            MaxTokens = _config.MaxTokens,
            Thinking = new ThinkingConfigDisabled(),
            System = system,
            Messages = messages,
            Tools = offered.Select(Describe).Select(tool => new ToolUnion(tool)).ToList()
        };
    }

    /// One of hers as the API wants it.
    ///
    /// The schema is passed through rather than rewritten. It came from a server that
    /// declared it, and a client that reshapes somebody else's schema is a client that
    /// will one day disagree with the server about what a tool accepts.
    private static WireTool Describe(Tools.Tool tool)
    {
        var properties = tool.Schema["properties"] is JsonObject declared
            ? declared.ToDictionary(
                property => property.Key,
                property => JsonSerializer.SerializeToElement(property.Value))
            : [];

        var required = tool.Schema["required"] is JsonArray names
            ? names.Select(name => name!.GetValue<string>()).ToList()
            : [];

        return new WireTool
        {
            Name = tool.Name,
            Description = tool.Description,
            InputSchema = new InputSchema { Properties = properties, Required = required }
        };
    }

    /// The accumulated fragments as an object.
    ///
    /// **Empty is the ordinary case, not a fault.** A tool that takes no arguments — which
    /// is four of the five UniFi ones — produces no `input_json_delta` at all, so nothing
    /// arrives and `{}` is the correct reading of it.
    private static JsonElement Arguments(StringBuilder json)
    {
        var text = json.ToString().Trim();
        if (text.Length == 0) text = "{}";

        try
        {
            return JsonDocument.Parse(text).RootElement.Clone();
        }
        catch (JsonException ex)
        {
            // Half a JSON object is not something to guess at. The registry answers an
            // unusable call with text, which is what everything else here does too.
            Log.Warn($"a tool call's arguments would not parse ({ex.Message}); treating it as empty");
            return JsonDocument.Parse("{}").RootElement.Clone();
        }
    }

    /// The same arguments as a field map, which is the shape the assistant echo wants.
    private static IReadOnlyDictionary<string, JsonElement> Fields(JsonElement arguments) =>
        arguments.ValueKind == JsonValueKind.Object
            ? arguments.EnumerateObject().ToDictionary(field => field.Name, field => field.Value.Clone())
            : new Dictionary<string, JsonElement>();

    public void Dispose() { }
}
```
