---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\BaselineChecks.cs
---

# tools\EarsTest\BaselineChecks.cs

```csharp
using Octavia.Core;
using Octavia.Rounds;

/// What she learns is normal, and when she is willing to say otherwise.
///
/// **These matter more than most.** For the first week she is silent by design, so a broken
/// baseline and a working one produce exactly the same experience for seven days — and the
/// first thing anyone would notice is either silence that should have been a warning, or a
/// warning about a torrent box that has been running all year.
internal static class BaselineChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail = "")
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed || detail.Length == 0 ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        var start = new DateTimeOffset(2026, 9, 1, 12, 0, 0, TimeSpan.Zero);

        // --- the learning week ------------------------------------------
        using (var scratch = new Scratch())
        {
            var baseline = new Baseline(scratch.Name, TimeSpan.FromDays(7));

            Check("before anything is seen, she is learning", baseline.Learning(start));

            /* A week of the owner's real network, in miniature: one machine that is always
               noisy. **Nothing said, whatever it looks like** - this is the whole point of the
               week, and the thing that would have gone wrong if severity had been the filter. */
            var said = 0;
            for (var hour = 0; hour < 24 * 7; hour++)
            {
                var counts = new Dictionary<string, int> { ["Proxmox-Host"] = 20 + hour % 7 };
                said += baseline.Observe(counts, start.AddHours(hour)).Count;
            }

            Check("a week of a noisy machine is said nothing about", said == 0, $"{said} said");
            Check("...and it was learned", baseline.Known == 1, $"{baseline.Known} known");
            Check("...and the week is over", !baseline.Learning(start.AddDays(7.1)));

            // The same machine, the same volume, after the week: still nothing.
            var after = baseline.Observe(
                new Dictionary<string, int> { ["Proxmox-Host"] = 24 }, start.AddDays(7.1));

            Check("the same machine afterwards is still not worth saying", after.Count == 0,
                  string.Join(", ", after));

            /* **The one that matters.** A machine that has not set off a single alert all week
               and now sets off four. Small in absolute terms, and the entire signal. */
            var stranger = baseline.Observe(
                new Dictionary<string, int> { ["Proxmox-Host"] = 22, ["Front-Door-Camera"] = 4 },
                start.AddDays(7.2));

            Check("a source never seen before is said", stranger.Count == 1, string.Join(", ", stranger));
            Check("...and it is named", stranger.FirstOrDefault()?.Contains("Front-Door-Camera") == true,
                  stranger.FirstOrDefault() ?? "nothing");
            Check("...and the noisy one is not dragged in with it",
                  stranger.All(s => !s.Contains("Proxmox")), string.Join(", ", stranger));

            // A known machine going far outside what it has ever done.
            var spike = baseline.Observe(
                new Dictionary<string, int> { ["Proxmox-Host"] = 400 }, start.AddDays(7.3));

            Check("a known source far above its usual is said", spike.Count == 1, string.Join(", ", spike));
            Check("...and says what usual was",
                  spike.FirstOrDefault()?.Contains("usually") == true, spike.FirstOrDefault() ?? "nothing");
        }

        // --- a flagged walk must not become the new normal ---------------
        using (var scratch = new Scratch())
        {
            var baseline = new Baseline(scratch.Name, TimeSpan.Zero);

            for (var hour = 0; hour < 20; hour++)
                baseline.Observe(new Dictionary<string, int> { ["known"] = 10 }, start.AddHours(hour));

            /* **A slow escalation must not teach her to accept it.** Four hours of something
               plainly wrong, each one worse than the last: if a flagged walk were folded into
               the baseline, the fourth would look ordinary and she would stop saying anything
               while it got worse. Every one of them has to be said. */
            var shouted = 0;
            for (var step = 1; step <= 4; step++)
                shouted += baseline.Observe(
                    new Dictionary<string, int> { ["known"] = 100 * step }, start.AddHours(20 + step)).Count;

            Check("a rising problem is said every time, not learned", shouted == 4, $"{shouted} of 4 said");
        }

        // --- nothing at all ----------------------------------------------
        using (var scratch = new Scratch())
        {
            var baseline = new Baseline(scratch.Name, TimeSpan.Zero);

            for (var hour = 0; hour < 20; hour++)
                baseline.Observe(new Dictionary<string, int> { ["known"] = 10 }, start.AddHours(hour));

            var quiet = baseline.Observe(new Dictionary<string, int>(), start.AddHours(21));

            // A quiet hour is not a finding. Whatever else this ever reports, it must never
            // wake somebody to say that nothing happened.
            Check("a quiet hour is not a finding", quiet.Count == 0, string.Join(", ", quiet));
        }

        // --- it survives a restart ----------------------------------------
        using (var scratch = new Scratch())
        {
            var first = new Baseline(scratch.Name, TimeSpan.FromDays(7));
            for (var hour = 0; hour < 30; hour++)
                first.Observe(new Dictionary<string, int> { ["known"] = 10 }, start.AddHours(hour));

            /* **She restarts a great deal** - a service pointed at a build directory restarts
               every time somebody compiles. A baseline that began again each time would never
               reach the end of its week, and would be silent for ever while looking healthy. */
            var second = new Baseline(scratch.Name, TimeSpan.FromDays(7));

            Check("a restart does not start the week again",
                  second.Began == start, $"began {second.Began}");
            Check("...and what was learned is still there", second.Known == 1, $"{second.Known} known");
        }

        // --- a corrupt file -----------------------------------------------
        using (var scratch = new Scratch())
        {
            File.WriteAllText(scratch.File, "{ this is not json");

            var baseline = new Baseline(scratch.Name, TimeSpan.FromDays(7));

            // A week of learning lost is annoying; refusing to start is worse, and judging
            // against half a file would be worst of all.
            Check("a corrupt baseline starts the learning again rather than throwing",
                  baseline.Learning(start) && baseline.Known == 0, $"{baseline.Known} known");
        }

        return failures;
    }

    /// A baseline file of its own, deleted afterwards. These write to her real data folder,
    /// so a name nothing else uses and a `finally` are the whole of the manners required.
    private sealed class Scratch : IDisposable
    {
        public string Name { get; } = $"check-{Guid.NewGuid():N}";

        public string File => Path.Combine(Paths.DataDir, $"baseline-{Name}.json");

        public void Dispose()
        {
            try { if (System.IO.File.Exists(File)) System.IO.File.Delete(File); }
            catch (Exception ex) { Console.WriteLine($"  ..   could not remove {File}: {ex.Message}"); }
        }
    }
}
```
