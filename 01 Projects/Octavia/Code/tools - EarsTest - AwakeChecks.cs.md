---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\AwakeChecks.cs
---

# tools\EarsTest\AwakeChecks.cs

```csharp
using Octavia.Senses;

/// The rule that decides whether the thing said *after* the wake phrase gets heard.
///
/// **v0.52.0 shipped this broken and nothing failed.** The window was measured from the wake
/// and lasted twelve seconds; a local brain took thirty-six to answer, so it expired while
/// she was still thinking, and the question that followed was dropped. The only symptom was a
/// log line blaming the wake word — which was working perfectly.
///
/// So these are written as the conversation they describe, in order, because that is the
/// shape the bug had: every individual piece behaved, and the sequence did not.
internal static class AwakeChecks
{
    public static int Run()
    {
        var failures = 0;
        void Check(string label, bool ok)
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}");
            if (!ok) failures++;
        }

        // Short enough that a check does not sit waiting, long enough that the machine being
        // busy cannot make it flake.
        var window = new AwakeWindow { For = TimeSpan.FromMilliseconds(400) };

        Check("she starts asleep", !window.IsOpen);

        window.Wake();
        Check("the phrase wakes her", window.IsOpen);

        /* The turn begins. **This is the thirty-six seconds** that used to lose the question:
           the brain is thinking, nothing is being said, and the clock was running out. */
        window.Hold(true);
        Thread.Sleep(600);
        Check("...and she is still awake after a slow brain has thought past the window",
              window.IsOpen);

        /* Each sentence she says re-arms it, so the window is measured from the end of what
           she said rather than the start. Held is still true here — she is mid-answer. */
        window.Wake();
        Check("a sentence she speaks keeps her awake", window.IsOpen);

        // The turn ends. Releasing the hold must *start* the clock, not stop it — the moment
        // she stops talking is the moment somebody is most likely to speak.
        window.Hold(false);
        Check("finishing her answer leaves her awake, not shut", window.IsOpen);

        Thread.Sleep(600);
        Check("...and she closes once nobody has said anything", !window.IsOpen);

        // A turn that dies must not leave her permanently awake, which would be the same bug
        // with the opposite sign: the wake word silently disabled instead of silently strict.
        window.Hold(true);
        window.Close();
        Check("an abandoned turn does not leave the wake word open for ever", !window.IsOpen);

        /* And the reason the number is what it is. The gate carries a follow-up for
           `GateFollowUpSeconds`; if this window were shorter, the audio would stop reaching
           Whisper before the gate's own rule ran out, and the tail of it would be dead code
           nobody could reach from a room. */
        var config = new Octavia.Core.OctaviaConfig();
        var armed = new AwakeWindow { For = TimeSpan.FromSeconds(Math.Max(12, config.GateFollowUpSeconds)) };
        Check($"the window is at least the gate's follow-up ({config.GateFollowUpSeconds}s)",
              armed.For.TotalSeconds >= config.GateFollowUpSeconds);

        return failures;
    }
}
```
