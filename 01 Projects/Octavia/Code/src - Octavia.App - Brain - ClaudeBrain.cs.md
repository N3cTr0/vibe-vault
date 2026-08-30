---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Brain\ClaudeBrain.cs
---

# src\Octavia.App\Brain\ClaudeBrain.cs

```csharp
using System.Runtime.CompilerServices;
using System.Text;
using Anthropic;
using Anthropic.Models.Messages;
using Octavia.Core;

namespace Octavia.Brain;

internal sealed class ClaudeBrain : IBrain
{
    private readonly OctaviaConfig _config;
    private readonly Conversation _history = new();
    private AnthropicClient? _client;
    private string? _keyInUse;

    public ClaudeBrain(OctaviaConfig config) => _config = config;

    public string Description => _config.Model;
    public bool IsReady => !string.IsNullOrWhiteSpace(SecretStore.ReadApiKey());
    public bool NeedsApiKey => true;

    public void Forget() => _history.Clear();

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

    public async IAsyncEnumerable<string> RespondAsync(
        string userText,
        Situation now = default,
        [EnumeratorCancellation] CancellationToken cancel = default)
    {
        // Resolve the client first: a missing key must not leave a dangling user turn
        // in the history, or the next request sends two user messages in a row.
        var client = Client();

        _history.Add(Utterance.User, userText);

        var parameters = new MessageCreateParams
        {
            Model = _config.Model,
            MaxTokens = _config.MaxTokens,
            Thinking = new ThinkingConfigDisabled(),
            System = new List<TextBlockParam>
            {
                new() { Text = Persona.System, CacheControl = new CacheControlEphemeral() }
            },
            Messages = _history.Turns
                .Select((t, i) => Turn(t, now, i == _history.Turns.Count - 1))
                .ToList()
        };

        var spoken = new StringBuilder();
        var pending = new StringBuilder();
        var failed = false;

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
                    if (enumerator.Current.TryPickContentBlockDelta(out var delta) &&
                        delta.Delta.TryPickText(out var text))
                    {
                        chunk = text.Text;
                    }
                }
                catch (OperationCanceledException)
                {
                    throw;
                }
                catch (Exception ex)
                {
                    Log.Write($"claude stream failed: {ex}");
                    failed = true;
                    break;
                }

                if (chunk is null) continue;

                spoken.Append(chunk);
                pending.Append(chunk);

                foreach (var sentence in Speech.DrainSentences(pending))
                    yield return sentence;
            }
        }
        finally
        {
            await enumerator.DisposeAsync();
        }

        var tail = pending.ToString().Trim();
        if (tail.Length > 0) yield return tail;

        if (failed && spoken.Length == 0)
        {
            _history.DropLast();
            throw new InvalidOperationException("The model could not be reached.");
        }

        _history.Add(Utterance.Assistant, spoken.ToString());
    }

    public void Dispose() { }
}
```
