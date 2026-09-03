---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\RoundChecks.cs
---

# tools\EarsTest\RoundChecks.cs

```csharp
using Octavia.Core;
using Octavia.Rounds;

/// Her rounds — the machinery behind speaking first.
///
/// **Every check here is about her staying quiet**, which is the point: a round whose normal
/// outcome is silence is indistinguishable from a broken one unless the silence is deliberate
/// and provable. The one check where she does speak exists mostly to prove the others are not
/// passing because nothing works at all.
internal static class RoundChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail = "")
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed || detail.Length == 0 ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        // --- a round that finds something, and one that never does -----
        var said = new List<string>();
        var config = Loud();

        using (var watchman = new Watchman(config, f => { said.Add(f.Sentence); return true; }))
        {
            watchman.Add(new Stub("quiet-one", null));
            await watchman.WalkAsync();

            Check("a round that finds nothing says nothing", said.Count == 0, $"{said.Count} said");
            Check("...and the walk is still recorded", watchman.LastWalk is not null, "no walk recorded");
            Check("...and says so in words", watchman.LastResult == "nothing to report", watchman.LastResult);
        }

        said.Clear();

        using (var watchman = new Watchman(config, f => { said.Add(f.Sentence); return true; }))
        {
            watchman.Add(new Stub("noisy", new Finding("Three events at high severity.", "the hourly check")));
            await watchman.WalkAsync();

            Check("a round that finds something says it", said.Count == 1, $"{said.Count} said");
            Check("...and says what it found",
                  said.FirstOrDefault() == "Three events at high severity.", said.FirstOrDefault() ?? "nothing");
        }

        // --- a round that throws must not stop the others ---------------
        said.Clear();

        using (var watchman = new Watchman(config, f => { said.Add(f.Sentence); return true; }))
        {
            watchman.Add(new Broken());
            watchman.Add(new Stub("after-the-broken-one", new Finding("The second round still ran.", "the hourly check")));
            await watchman.WalkAsync();

            /* A source being unreachable is not a threat and must never be announced as one —
               and it must not take the other rounds down with it. Both halves matter: the
               first is honesty, the second is that one flaky integration would otherwise
               silence every other thing she watches. */
            Check("a round that throws is not reported as a finding",
                  said.All(s => !s.Contains("could not")), string.Join(" / ", said));
            Check("...and the rounds after it still walk", said.Count == 1, $"{said.Count} said");
        }

        // --- quiet hours -------------------------------------------------
        var night = Loud();
        night.Rounds.QuietFrom = "22:30";
        night.Rounds.QuietTo = "07:30";

        said.Clear();

        using (var watchman = new Watchman(night, f => { said.Add(f.Sentence); return true; }))
        {
            watchman.Add(new Once("nocturnal", new Finding("Something at four in the morning.", "the hourly check")));

            // Four in the morning, stated rather than waited for.
            await watchman.WalkAsync(At(4, 0));

            Check("nothing is said during quiet hours", said.Count == 0, string.Join(" / ", said));
            Check("...but it is held rather than dropped",
                  watchman.LastResult.Contains("held"), watchman.LastResult);

            /* **The whole reason to check at four in the morning is that somebody hears about
               it at eight.** The round finds nothing on this second walk, so anything said
               here came from the first one. */
            await watchman.WalkAsync(At(8, 0));

            Check("...and said once the quiet ends", said.Count == 1, $"{said.Count} said");
            Check("...with how long ago it happened",
                  said.FirstOrDefault()?.StartsWith("4 hours ago") == true, said.FirstOrDefault() ?? "nothing");
        }

        // A window of zero length is how quiet hours are switched off, rather than a magic
        // empty string that has to be remembered.
        var never = Loud();
        never.Rounds.QuietFrom = never.Rounds.QuietTo = "08:00";

        using (var watchman = new Watchman(never, _ => true))
        {
            Check("equal quiet hours mean no quiet hours",
                  !watchman.Quiet(new DateTimeOffset(2026, 9, 3, 3, 0, 0, TimeSpan.Zero)), "3am read as quiet");
        }

        // The window everybody actually configures is the one that crosses midnight, and it
        // is the one an obvious `>= from && < to` gets wrong in both directions.
        var overnight = Loud();
        overnight.Rounds.QuietFrom = "22:30";
        overnight.Rounds.QuietTo = "07:30";

        using (var watchman = new Watchman(overnight, _ => true))
        {
            Check("a window over midnight is quiet before midnight",
                  watchman.Quiet(At(23, 15)), "23:15 read as awake");
            Check("...and after midnight", watchman.Quiet(At(3, 0)), "03:00 read as awake");
            Check("...and not in the afternoon", !watchman.Quiet(At(15, 0)), "15:00 read as quiet");
            Check("...and not one minute after it ends", !watchman.Quiet(At(7, 31)), "07:31 read as quiet");
        }

        /* **`24:00` is what somebody types when they mean midnight**, and it is not a time
           `TimeOnly` will parse. The owner typed exactly that. Rejecting it would be
           defensible and useless — a config file is not a standards body. */
        var midnight = Loud();
        midnight.Rounds.QuietFrom = "24:00";
        midnight.Rounds.QuietTo = "08:30";

        using (var watchman = new Watchman(midnight, _ => true))
        {
            Check("24:00 is read as midnight", watchman.Quiet(At(2, 0)), "02:00 read as awake");
            Check("...through to the end of the window", watchman.Quiet(At(8, 29)), "08:29 read as awake");
            Check("...and not one minute after", !watchman.Quiet(At(8, 31)), "08:31 read as quiet");
            Check("...and not in the evening", !watchman.Quiet(At(21, 0)), "21:00 read as quiet");
        }

        /* A time that cannot be read used to fall through to *never quiet*, silently — the
           most expensive possible reading of a typo, and indistinguishable from a correct
           file. It still errs towards being awake, because a companion that will not speak is
           worse than one that speaks at the wrong hour, but it says so now. */
        var nonsense = Loud();
        nonsense.Rounds.QuietFrom = "half past ten";
        nonsense.Rounds.QuietTo = "08:30";

        using (var watchman = new Watchman(nonsense, _ => true))
        {
            Check("an unreadable quiet hour does not silently mean 'never quiet'",
                  !watchman.Quiet(At(2, 0)), "it read as quiet, which hides the typo differently");
        }

        // --- she is busy ---------------------------------------------------
        said.Clear();
        var asked = 0;

        var busy = Loud();

        using (var watchman = new Watchman(busy, f => { asked++; if (asked < 2) return false; said.Add(f.Sentence); return true; }))
        {
            watchman.Add(new Stub("patient", new Finding("It waited its turn.", "the hourly check")));

            var walking = watchman.WalkAsync();
            var finished = await Task.WhenAny(walking, Task.Delay(TimeSpan.FromSeconds(40)));

            Check("a finding waits rather than interrupting", finished == walking, "it never gave up or delivered");
            Check("...and is said once she is free", said.Count == 1, $"{said.Count} said after {asked} attempts");
        }

        // --- switched off ----------------------------------------------------
        var off = Loud();
        off.Rounds.Enabled = false;

        using (var watchman = new Watchman(off, _ => { failures++; return true; }))
        {
            watchman.Add(new Stub("should-not-run", new Finding("This must never be said.", "the hourly check")));
            watchman.Start();
            await Task.Delay(200);

            Check("switched off, she does not walk at all", watchman.LastWalk is null, "it walked anyway");
        }

        return failures;
    }

    private static DateTimeOffset At(int hour, int minute) =>
        new(2026, 9, 3, hour, minute, 0, TimeSpan.Zero);

    /// **No quiet hours at all**, which is what an equal pair means.
    ///
    /// It matters that this is not the default 22:30-07:30: a suite run late at night would
    /// then find every "she says it" check failing, correctly, for a reason that has nothing
    /// to do with what is being tested. A check that passes or fails by the clock is worse
    /// than no check.
    private static OctaviaConfig Loud() => new()
    {
        Rounds = new RoundsConfig { Enabled = true, EveryMinutes = 60, QuietFrom = "00:00", QuietTo = "00:00" }
    };

    /// Finds the same thing every walk.
    private sealed class Stub(string name, Finding? finding) : IRound
    {
        public string Name => name;
        public Task<Finding?> CheckAsync(CancellationToken cancel) => Task.FromResult(finding);
    }

    /// Finds something once and nothing after, so a second walk proves that what was said
    /// came out of the hold rather than from checking again.
    private sealed class Once(string name, Finding finding) : IRound
    {
        private bool _spent;

        public string Name => name;

        public Task<Finding?> CheckAsync(CancellationToken cancel)
        {
            if (_spent) return Task.FromResult<Finding?>(null);
            _spent = true;
            return Task.FromResult<Finding?>(finding);
        }
    }

    private sealed class Broken : IRound
    {
        public string Name => "broken";
        public Task<Finding?> CheckAsync(CancellationToken cancel) =>
            throw new HttpRequestException("the gateway could not be reached");
    }
}
```
