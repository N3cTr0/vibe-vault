---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\AttentionGate.cs
---

# src\Octavia.Core\Brain\AttentionGate.cs

```csharp
using System.Net.Http.Json;
using System.Text.Json;
using Octavia.Core;

namespace Octavia.Brain;

/// <param name="Answer">Whether this deserves a reply.</param>
/// <param name="Why">In a few words, for the log. "She ignored me" has to be answerable.</param>
/// <param name="Cost">What deciding took, so the gate can be shown to be cheaper than what it saves.</param>
internal readonly record struct Verdict(bool Answer, string Why, TimeSpan Cost);

/// Decides whether something she heard was addressed to her, and worth answering.
///
/// This is the component Stage 2 was really building when it added a local model: a room
/// microphone hears the television, the other end of a phone call, and both halves of a
/// conversation she is not in. Sending all of that to Claude is what makes always-on
/// listening unaffordable, and answering it is what makes her insufferable.
///
/// Two layers, cheapest first. Rules settle the clear cases in microseconds; only the
/// genuinely ambiguous ones cost a model call, and that call goes to a small local model
/// which is free. **No paid model is ever used to decide whether to use a paid model.**
internal sealed class AttentionGate : IDisposable
{
    private readonly OctaviaConfig _config;
    private readonly HttpClient _http;
    private DateTime _lastExchange = DateTime.MinValue;

    public AttentionGate(OctaviaConfig config)
    {
        _config = config;
        _http = new HttpClient
        {
            BaseAddress = new Uri(config.LocalEndpoint.TrimEnd('/') + "/"),
            // Generous, once the free layer is understood: anything named or in an
            // ongoing conversation never reaches here, so this latency falls almost
            // entirely on traffic that is about to be *rejected* — where nobody is
            // waiting for an answer. Four seconds was too tight on a loaded machine and
            // bought nothing but timeouts, each of which then failed open and paid for
            // the call anyway.
            Timeout = TimeSpan.FromSeconds(8)
        };
    }

    public bool Enabled => !string.Equals(_config.Gate, "off", StringComparison.OrdinalIgnoreCase);

    public string Description => Enabled ? $"{_config.GateModel} (local)" : "off";

    /// Called when a turn completes, so the next few seconds count as the same conversation.
    public void Answered() => _lastExchange = DateTime.UtcNow;

    /// True while a follow-up would still be understood as part of the exchange.
    private bool StillTalking =>
        (DateTime.UtcNow - _lastExchange).TotalSeconds < _config.GateFollowUpSeconds;

    public async Task<Verdict> JudgeAsync(string heard, CancellationToken cancel = default)
    {
        if (!Enabled) return new Verdict(true, "gate off", TimeSpan.Zero);

        var started = DateTime.UtcNow;
        var text = heard.Trim();

        // ---- the free layer -------------------------------------

        // Her name is the one unambiguous signal there is. Nothing that says it should
        // ever be dropped, whatever a model thinks of the rest of the sentence.
        if (Names().Any(name => text.Contains(name, StringComparison.OrdinalIgnoreCase)))
            return new Verdict(true, "named", DateTime.UtcNow - started);

        // Mid-conversation, the name is not said again — "and what about tomorrow?" is
        // addressed to her by context alone. Without this she is unusable past one turn.
        if (StillTalking)
            return new Verdict(true, "follow-up", DateTime.UtcNow - started);

        // Too short to carry an address or a request. The recogniser's own floor is
        // lower, because a short utterance is still worth *transcribing*.
        if (text.Length < 12)
            return new Verdict(false, "too short to be addressed to anyone", DateTime.UtcNow - started);

        // ---- the model layer ------------------------------------

        try
        {
            var answer = await AskAsync(text, cancel);
            return new Verdict(answer.Yes, answer.Why, DateTime.UtcNow - started);
        }
        catch (Exception ex)
        {
            // Fails open, deliberately. A companion who goes silent because a helper
            // model stopped answering is broken; one who occasionally replies to the
            // television is merely annoying. The log says which happened.
            Log.Warn($"gate unavailable, answering anyway: {ex.Message}");
            return new Verdict(true, "gate unavailable", DateTime.UtcNow - started);
        }
    }

    /// What she answers to. Configurable because "Octavia" is what *this* one is called.
    private IEnumerable<string> Names() =>
        _config.WakeNames.Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

    /// Short on purpose, twice over.
    ///
    /// Every token here is processed on every call, and on a modest machine the prompt
    /// dominates the latency — halving this halved the median.
    ///
    /// And the first version asked whether the line was "addressed to an assistant",
    /// which sounds right and is wrong: the model read *addressed* as *named*, and threw
    /// away "tell me a short joke" and "can you turn the volume down". People do not say
    /// her name every time. The question that works is whether it is a request at all.
    private const string Instruction = """
        A microphone is always on in someone's home. Decide whether this line is a
        question or a request that an assistant should answer.

        YES: questions, requests and instructions, even with no name in them.
        NO: television, lyrics, one side of a phone call, people talking to each other,
        muttering, fragments.

        Reply YES or NO, then a dash and at most four words. Nothing else.
        """;

    private async Task<(bool Yes, string Why)> AskAsync(string text, CancellationToken cancel)
    {
        var request = new
        {
            model = _config.GateModel,
            messages = new object[]
            {
                new { role = "system", content = Instruction },
                new { role = "user", content = text }
            },
            stream = false,
            temperature = 0,
            max_tokens = 24
        };

        // No switch is sent to suppress a reasoning model's scratchpad. There isn't a
        // portable one — `think`, `/no_think` and `chat_template_kwargs` were each tried
        // against Ollama's OpenAI-compatible endpoint and each was ignored, leaving the
        // model to spend all 24 tokens deliberating and return an empty answer. The fix
        // is to not use a reasoning model as a gate; see GateModel in the config.

        using var response = await _http.PostAsJsonAsync("chat/completions", request, cancel);
        response.EnsureSuccessStatusCode();

        var body = await response.Content.ReadFromJsonAsync<JsonElement>(cancel);
        var reply = body.GetProperty("choices")[0].GetProperty("message")
                        .GetProperty("content").GetString() ?? "";

        return Read(reply);
    }

    /// Small models do not reliably answer in the shape they were asked to. This reads
    /// the answer out of whatever came back rather than trusting the format, and treats
    /// an unreadable reply as a yes — the same fail-open rule as an unreachable server.
    internal static (bool Yes, string Why) Read(string reply)
    {
        var text = Speech.WithoutThinking(reply).Trim();
        if (text.Length == 0) return (true, "gate said nothing");

        var dash = text.IndexOfAny(['-', '—', ':']);
        var head = (dash > 0 ? text[..dash] : text).Trim();
        var why = dash > 0 && dash + 1 < text.Length ? text[(dash + 1)..].Trim() : "";

        // A trailing "NO" inside the reason must not outvote the verdict, so only the
        // first word is read as the answer.
        var first = head.Split([' ', ',', '.', '\n'], StringSplitOptions.RemoveEmptyEntries)
                        .FirstOrDefault() ?? "";

        if (first.StartsWith("NO", StringComparison.OrdinalIgnoreCase))
            return (false, why.Length > 0 ? why : "not addressed to her");

        if (first.StartsWith("YES", StringComparison.OrdinalIgnoreCase))
            return (true, why.Length > 0 ? why : "addressed to her");

        return (true, "gate was unclear");
    }

    public void Dispose() => _http.Dispose();
}
```
