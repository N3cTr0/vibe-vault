---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Rounds\Watchman.cs
---

# src\Octavia.Core\Rounds\Watchman.cs

```csharp
using Octavia.Core;

namespace Octavia.Rounds;

/// The clock behind her rounds, and the manners around them.
///
/// Named for what it is rather than what it does: something that walks the same route on the
/// hour, usually finds nothing, and is worth having precisely for the nights it does.
///
/// **Four rules, and each one exists because the obvious version is worse:**
///
/// | | |
/// |---|---|
/// | It never interrupts | She is answering somebody. A finding waits, and says how long it waited. |
/// | It is quiet at night | An hourly job that can talk is an hourly job that can wake the house. |
/// | Silence is recorded | *"I checked and there was nothing"* is the normal outcome, and an unfired round looks exactly like a working one unless it says so. |
/// | It never asks the model | The round's own data decides whether to speak. See `Finding`. |
internal sealed class Watchman : IDisposable
{
    /// How long a finding will wait for her to stop talking before it gives up. Long enough
    /// to outlast an ordinary exchange, short enough that nothing an hour old is announced
    /// as though it just happened.
    private static readonly TimeSpan Patience = TimeSpan.FromMinutes(5);

    private readonly OctaviaConfig _config;
    private readonly List<IRound> _rounds = [];
    private readonly CancellationTokenSource _stopping = new();

    /// Delivers a finding, or answers false because she is mid-turn. Deliberately a function
    /// rather than an event: the answer matters, and an event has nowhere to put one.
    private readonly Func<Finding, bool> _deliver;

    /// Findings that happened while nobody should be spoken to. Reported together when the
    /// quiet ends, rather than dropped — the whole point of checking at four in the morning
    /// is that somebody hears about it at eight.
    private readonly List<(DateTimeOffset When, Finding What)> _held = [];

    private Task? _walking;

    public Watchman(OctaviaConfig config, Func<Finding, bool> deliver)
    {
        _config = config;
        _deliver = deliver;
    }

    public void Add(IRound round) => _rounds.Add(round);

    /// What the last walk found, for the health panel. Silence has to be legible somewhere.
    public string LastResult { get; private set; } = "not walked yet";
    public DateTimeOffset? LastWalk { get; private set; }

    public void Start()
    {
        /* Said out loud rather than returned from in silence. An empty route is the correct
           state today — no round is written yet — but "she has nothing to check" and "her
           rounds are broken" produce exactly the same experience, which is the failure this
           whole class is arranged around. It should not be exempt from its own rule. */
        if (_rounds.Count == 0)
        {
            Log.Write("rounds: none registered, so nothing is walked");
            return;
        }

        if (!_config.Rounds.Enabled)
        {
            Log.Write($"rounds: {_rounds.Count} registered but switched off in config");
            return;
        }

        _walking = Task.Run(WalkForeverAsync);
        Log.Write($"rounds: every {_config.Rounds.EveryMinutes} min " +
                  $"({string.Join(", ", _rounds.Select(r => r.Name))}); quiet {_config.Rounds.QuietFrom}-{_config.Rounds.QuietTo}");
    }

    private async Task WalkForeverAsync()
    {
        var every = TimeSpan.FromMinutes(Math.Clamp(_config.Rounds.EveryMinutes, 1, 24 * 60));

        /* One interval before the first walk, rather than one immediately.
           A server that restarts a few times while somebody is working on it would otherwise
           announce the same finding at every start — and the first thing she should do on
           coming up is not talk. */
        while (!_stopping.IsCancellationRequested)
        {
            try { await Task.Delay(every, _stopping.Token); }
            catch (OperationCanceledException) { return; }

            try { await WalkAsync(); }
            catch (Exception ex) { Log.Error("a round threw", ex); }
        }
    }

    /// Says what a finding sounds like, through the real path, on request.
    ///
    /// **Everything about this is the genuine article except the finding itself**: the same
    /// delivery, the same room, the same voice, the same entry in the room's history. That is
    /// the point — an hourly errand is a thing nobody sees working until the night it matters,
    /// and *"what will she actually do"* is a fair question to be able to answer on a Tuesday.
    ///
    /// **The sentence says it is a rehearsal**, and that is not decoration: it goes into her
    /// history like any other turn, and a line there claiming a camera was attacked would be
    /// read as fact by her the next time somebody asked about it.
    ///
    /// It does not touch the baseline, and it ignores quiet hours — a person asked for it, and
    /// they are plainly awake.
    public string Rehearse()
    {
        var finding = new Finding(
            "This is a rehearsal, not a real alert. If something new turned up on the network I would " +
            "tell you like this. Something new: the front door camera, four blocked intrusion attempts " +
            "in the last hour, where it has had none all week.",
            "(a rehearsal of her hourly security round, asked for from the dev panel — not a real finding)");

        if (!_deliver(finding))
        {
            Log.Write("rounds: a rehearsal was asked for while she was mid-turn");
            return "She is in the middle of something; ask again in a moment.";
        }

        Log.Write("rounds: rehearsed a finding");
        return "That is what a finding sounds like.";
    }

    /// One walk of every round. Public so a check can drive it without waiting an hour.
    ///
    /// `now` is injectable for the same reason, and only that reason. **A quiet window cannot
    /// express "always"** — every wrap-around window covers 24 hours minus a minute — so a
    /// check that wanted to prove a finding is *held* overnight would have had a one-minute
    /// hole in it, and a check that fails once a day is worse than no check. Passing the hour
    /// in makes it exact.
    public async Task WalkAsync(DateTimeOffset? now = null)
    {
        var at = now ?? DateTimeOffset.Now;

        var found = new List<Finding>();

        foreach (var round in _rounds)
        {
            if (_stopping.IsCancellationRequested) return;

            try
            {
                if (await round.CheckAsync(_stopping.Token) is { } finding) found.Add(finding);
            }
            catch (Exception ex)
            {
                // A source that is down is not a threat, and must not be reported as one -
                // nor may it stop the other rounds walking.
                Log.Warn($"round '{round.Name}' could not check: {ex.Message}");
            }
        }

        LastWalk = at;

        if (found.Count == 0 && _held.Count == 0)
        {
            LastResult = "nothing to report";
            Log.Write($"rounds: walked {_rounds.Count}, nothing to report");
            return;
        }

        if (Quiet(at))
        {
            foreach (var finding in found) _held.Add((at, finding));
            LastResult = $"{_held.Count} held until the quiet hours end";
            Log.Write($"rounds: {found.Count} found, {_held.Count} held - quiet hours");
            return;
        }

        foreach (var finding in Due(found, at)) Deliver(finding);
    }

    /// Anything held overnight first, oldest first, each said with how long ago it happened —
    /// then whatever this walk turned up.
    private IEnumerable<Finding> Due(List<Finding> found, DateTimeOffset at)
    {
        foreach (var (when, what) in _held.OrderBy(h => h.When))
            yield return what with { Sentence = $"{Ago(at - when)}, {Lower(what.Sentence)}" };

        _held.Clear();

        foreach (var finding in found) yield return finding;
    }

    private void Deliver(Finding finding)
    {
        var until = DateTimeOffset.Now + Patience;

        while (DateTimeOffset.Now < until)
        {
            if (_deliver(finding))
            {
                LastResult = finding.Sentence;
                Log.Write($"rounds: said it - {finding.Sentence}");
                return;
            }

            // She is mid-turn. Waiting is the whole point: cutting across somebody to
            // announce a port scan is worse than the port scan.
            if (_stopping.Token.WaitHandle.WaitOne(TimeSpan.FromSeconds(10))) return;
        }

        LastResult = $"could not be said - she was busy: {finding.Sentence}";
        Log.Warn($"rounds: gave up waiting to say it - {finding.Sentence}");
    }

    /// True when nobody should be spoken to. Handles a window that crosses midnight, which is
    /// the only kind anybody actually configures.
    public bool Quiet(DateTimeOffset now)
    {
        if (Hour(_config.Rounds.QuietFrom) is not { } from || Hour(_config.Rounds.QuietTo) is not { } to)
        {
            /* **Said out loud rather than silently meaning "never quiet".**

               A misspelled time used to fall straight through to `false`, which is the most
               expensive possible reading of it: somebody who typed `24:00` meaning midnight
               would have got a companion that talks at four in the morning and a config file
               that looks exactly right. Once per walk is the correct volume for this — it is
               a standing fault, not an event. */
            Log.Warn($"rounds: quiet hours '{_config.Rounds.QuietFrom}' to '{_config.Rounds.QuietTo}' " +
                     "cannot be read as times, so she is treating every hour as awake");
            return false;
        }

        if (from == to) return false;

        var at = TimeOnly.FromDateTime(now.DateTime);

        return from < to
            ? at >= from && at < to          // 01:00 to 06:00
            : at >= from || at < to;         // 22:30 to 07:30, over midnight
    }

    /// A time of day, accepting the one people actually write for midnight.
    ///
    /// **`24:00` is not a time `TimeOnly` will parse**, and it is what somebody types when
    /// they mean "from midnight" — the owner did, asking for quiet hours of 24:00 to 08:30.
    /// Rejecting it would be defensible and useless; a config file is not a standards body.
    /// `24:00` and `00:00` are the same instant, so it is read as one.
    private static TimeOnly? Hour(string? text)
    {
        if (string.IsNullOrWhiteSpace(text)) return null;

        var trimmed = text.Trim();
        if (trimmed is "24:00" or "24:00:00" or "24") return TimeOnly.MinValue;

        return TimeOnly.TryParse(trimmed, System.Globalization.CultureInfo.InvariantCulture, out var parsed)
            ? parsed
            : null;
    }

    private static string Ago(TimeSpan span) => span.TotalMinutes switch
    {
        < 90 => "a little while ago",
        < 60 * 5 => $"{(int)span.TotalHours} hours ago",
        _ => "overnight"
    };

    /// So "Three events..." reads properly after "Overnight, ...". Only the first letter, and
    /// only when the second one is lower case — otherwise `UDM` and `IPS` get mangled.
    private static string Lower(string sentence) =>
        sentence.Length > 1 && char.IsUpper(sentence[0]) && !char.IsUpper(sentence[1])
            ? char.ToLowerInvariant(sentence[0]) + sentence[1..]
            : sentence;

    public void Dispose()
    {
        _stopping.Cancel();
        try { _walking?.Wait(TimeSpan.FromSeconds(2)); } catch (Exception ex) { Log.Debug($"rounds stop: {ex.Message}"); }
        _stopping.Dispose();
    }
}
```
