---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\GateChecks.cs
---

# tools\EarsTest\GateChecks.cs

```csharp
// The gate decides what reaches a paid model, so both of its failure modes are
// expensive in different currencies: too tight and she ignores you, too loose and she
// answers the television at Anthropic's prices.
//
// The rules layer and the reply parser are pure and are asserted. Whether the *model*
// judges well is a matter of taste on real sentences, so that part is a probe: it
// prints a scored table against a labelled corpus and leaves the reading to a person.
using Octavia.Brain;
using Octavia.Core;

internal static class GateChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        // --- reading a small model's answer ------------------------
        // Small models do not answer in the shape they were asked to, so this is read
        // defensively. Every case below is a reply shape one of them actually produced.
        var readings = new (string Reply, bool Yes, string Note)[]
        {
            ("YES - a question",              true,  "the documented shape"),
            ("NO - television dialogue",      false, "the documented shape"),
            ("no — someone else talking",     false, "lowercase, em dash"),
            ("YES: asking for the time",      true,  "colon instead of a dash"),
            ("NO",                            false, "verdict alone"),
            ("<think>hmm</think>NO - media",  false, "a reasoning model that did not obey think=false"),
            ("NO, this is not addressed",     false, "a comma, and the word NO again later"),
            ("YES - answer, do not say NO",   true,  "a NO inside the reason must not outvote YES"),
            ("",                              true,  "nothing at all fails open"),
            ("perhaps?",                      true,  "unreadable fails open"),
            ("<think>unfinished",             true,  "ran out of tokens mid-thought, fails open"),
        };

        foreach (var (reply, expected, note) in readings)
        {
            var (yes, _) = AttentionGate.Read(reply);
            Check($"reads {note}", yes == expected, $"'{reply}' read as {(yes ? "YES" : "NO")}");
        }

        // --- the free layer ----------------------------------------
        var config = new OctaviaConfig { Gate = "local", WakeNames = "Octavia, Octi" };
        using var gate = new AttentionGate(config);

        var named = gate.JudgeAsync("so anyway octavia what is the time").Result;
        Check("her name is always let through", named.Answer && named.Why == "named", named.Why);

        var alias = gate.JudgeAsync("octi are you there").Result;
        Check("a second name works too", alias.Answer && alias.Why == "named", alias.Why);

        var fragment = gate.JudgeAsync("yeah no").Result;
        Check("a fragment is not addressed to anyone", !fragment.Answer, fragment.Why);

        // The name check must not need a model, or an unreachable server would swallow
        // the one signal that is never ambiguous.
        Check("the free layer costs nothing", named.Cost.TotalMilliseconds < 50,
            $"{named.Cost.TotalMilliseconds:0} ms");

        // --- follow-up ---------------------------------------------
        var cold = gate.JudgeAsync("and what about tomorrow").Result;
        gate.Answered();
        var warm = gate.JudgeAsync("and what about tomorrow").Result;

        Check("a follow-up is let through after she answers", warm.Answer && warm.Why == "follow-up", warm.Why);
        Check("the same words are judged when she has not", cold.Why != "follow-up", cold.Why);

        // --- off ---------------------------------------------------
        using var open = new AttentionGate(new OctaviaConfig { Gate = "off" });
        var ungated = open.JudgeAsync("mumble mumble").Result;
        Check("off lets everything through", ungated.Answer && ungated.Why == "gate off", ungated.Why);

        // --- when she opens her eyes -------------------------------
        // Deliberately asymmetric. Failing to look means she says she cannot see;
        // looking when nobody asked means a camera turned on in someone's home. Every
        // case below is checked in the direction that matters for that mistake.
        var needsEyes = new[]
        {
            "can you see what I'm holding",
            "look at this for me",
            "what colour is this jumper",
            "what does this say",
            "how do i look",
            "what am i wearing",
            "take a look at the screen"
        };

        foreach (var text in needsEyes)
            Check($"looks for \"{text}\"", Sight.WantsEyes(text), "did not want eyes");

        var doesNot = new[]
        {
            "what is the weather doing tomorrow",
            "tell me a short joke",
            "am i boring you",                      // "am i", with nothing to look at
            "i can see why that would be annoying", // "see", but not asking her to
            "set a timer for ten minutes",
            "what time is it"
        };

        foreach (var text in doesNot)
            Check($"stays shut for \"{text}\"", !Sight.WantsEyes(text), "wanted eyes");

        return failures;
    }
}
```
