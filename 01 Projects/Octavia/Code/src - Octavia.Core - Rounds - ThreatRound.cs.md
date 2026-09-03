---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Rounds\ThreatRound.cs
---

# src\Octavia.Core\Rounds\ThreatRound.cs

```csharp
using Octavia.Brain.Tools;
using Octavia.Core;

namespace Octavia.Rounds;

/// The first thing she checks on her own: the gateway's security log.
///
/// It reads through her own tool rather than talking to the gateway itself, which is worth a
/// sentence. The login, the session, the paging and the grouping all live in
/// `tools\unifi-mcp.ps1` already, and putting a second copy here would mean two things to keep
/// right — and would leave her unable to *answer* about threats in conversation, which she can,
/// because it is a tool like any other.
///
/// **It judges nothing itself.** The tool reports and `Baseline` decides what is unusual; this
/// only turns a departure into a sentence. That split is the point: the interesting question is
/// never "were there threats" — on the network this was built against there are always about
/// twenty-four an hour — but "is this the same as it has been".
internal sealed class ThreatRound(ToolRegistry tools, OctaviaConfig config) : IRound
{
    private const string Tool = "unifi__recent_threats";

    private readonly Baseline _baseline = new("threats", TimeSpan.FromDays(Math.Clamp(config.Rounds.LearnForDays, 0, 90)));

    private DateTimeOffset _since = DateTimeOffset.UtcNow;

    public string Name => "threats";

    /// For the health panel: whether she is still learning, and how much longer.
    public string Describe(DateTimeOffset now) => _baseline.Describe(now);

    public async Task<Finding?> CheckAsync(CancellationToken cancel)
    {
        var now = DateTimeOffset.Now;

        // From the end of the last walk, not from a fixed hour back: a walk that was late, or a
        // server that was asleep, must not leave a gap nobody ever looks at.
        var since = _since;
        _since = DateTimeOffset.UtcNow;

        var arguments = System.Text.Json.JsonDocument.Parse(
            $$"""{"format":"counts","since":{{since.ToUnixTimeMilliseconds()}}}""").RootElement;

        var answer = await tools.CallAsync(Tool, arguments, cancel: cancel);

        /* A tool that is missing or unreachable is **not** a finding. The registry answers with
           a sentence rather than throwing, so this checks the shape it expects instead of
           trusting that anything came back — a gateway that is down would otherwise be
           announced as a security event, which is precisely backwards. */
        var (total, sources) = Parse(answer.Text);

        if (sources is null)
        {
            Log.Warn($"round 'threats': {Tool} did not answer with counts: {First(answer.Text)}");
            return null;
        }

        /* `total` is a summary line and **must not reach the baseline as a source**. Left in,
           it would be learned as the busiest "client" on the network, and every quiet hour
           after a busy one would read as that client having gone away. It is pulled out in
           `Parse` for exactly that reason. */
        var departures = _baseline.Observe(sources, now);
        if (departures.Count == 0) return null;

        var what = departures.Count == 1
            ? departures[0]
            : string.Join(", ", departures.Take(3)) + (departures.Count > 3 ? $" and {departures.Count - 3} more" : "");

        return new Finding(
            $"Something new on the network: {what}. That is not what the last while has looked like. " +
            $"{total} security event(s) in the last hour, all blocked.",
            $"(her hourly check of the security log found {departures.Count} source(s) departing from " +
            $"what she has learned is normal: {string.Join("; ", departures)})");
    }

    /// `name<TAB>number` per line, which is what the tool's `counts` format promises.
    ///
    /// Anything that does not parse as that is **no answer at all** rather than zero. The two
    /// are very different and only one of them should teach her that a quiet hour happened: a
    /// gateway that is down, a session that expired, a tool that was renamed — every one of
    /// those looks like silence, and silence learned as normal is how a baseline rots.
    private static (int Total, Dictionary<string, int>? Sources) Parse(string text)
    {
        if (string.IsNullOrWhiteSpace(text)) return (0, null);

        var sources = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
        var total = 0;
        var sawTotal = false;

        foreach (var line in text.Split('\n', StringSplitOptions.RemoveEmptyEntries))
        {
            var parts = line.Split('\t');
            if (parts.Length != 2 || !int.TryParse(parts[1].Trim(), out var count)) return (0, null);

            var key = parts[0].Trim();

            if (key.Equals("total", StringComparison.OrdinalIgnoreCase))
            {
                total = count;
                sawTotal = true;
                continue;
            }

            sources[key] = count;
        }

        // The total line is the one that is always present, including on a quiet hour with no
        // sources at all. Its absence means this was not a `counts` answer.
        return sawTotal ? (total, sources) : (0, null);
    }

    private static string First(string text) =>
        text.Split('\n')[0] is { Length: > 100 } long_ ? long_[..100] + "..." : text.Split('\n')[0];
}
```
