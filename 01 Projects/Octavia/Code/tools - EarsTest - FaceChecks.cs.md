---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\FaceChecks.cs
---

# tools\EarsTest\FaceChecks.cs

```csharp
// The two host-side halves of Stage 5: which mouth shape a phoneme becomes, and which
// mood a sentence carries. Both feed the face every few hundred milliseconds while she
// talks, and both are wrong in a way nobody notices until the character looks odd.
using Octavia.Brain;
using Octavia.Voice;

internal static class FaceChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        /* **The SAPI viseme checks went with SAPI**, v0.37.0.

           They asserted a mapping from Windows' 21 viseme ids onto the five VRM mouth
           shapes, and that mapping existed only inside `SapiVoice`. The neural voice does not
           receive viseme ids at all — `VisemeReader` derives the mouth from the waveform,
           which is why her mouth stays in step with audio that is streamed rather than
           played.

           Deleted rather than ported: there is no id to map any more, so a rewritten version
           would be asserting arithmetic nothing performs. What replaced them in substance is
           the viseme half of the face-protocol checks, which watch the shapes she actually
           sends. */

        // --- sentence -> expression ----------------------------------
        var presets = new[] { "neutral", "happy", "angry", "sad", "relaxed", "surprised" };

        Check("an apology reads as sad", Moods.Read("I'm sorry, that didn't work.").Name == "sad",
            Moods.Read("I'm sorry, that didn't work.").Name);
        Check("a pleasantry reads as happy", Moods.Read("I'd be glad to.").Name == "happy",
            Moods.Read("I'd be glad to.").Name);
        Check("surprise is recognised", Moods.Read("Wow, that is a lot.").Name == "surprised",
            Moods.Read("Wow, that is a lot.").Name);
        Check("a plain statement stays neutral", Moods.Read("The kettle is on the counter.").Name == "neutral",
            Moods.Read("The kettle is on the counter.").Name);
        Check("empty text stays neutral", Moods.Read("").Name == "neutral", "not neutral");
        Check("neutral carries no weight", Moods.Read("A plain sentence.").Weight == 0, "weight is not zero");
        Check("punctuation colours only lightly", Moods.Read("Here you go!").Weight <= 0.45,
            Moods.Read("Here you go!").Weight.ToString("0.00"));

        // Anything the face cannot name would silently fall back to neutral, which is
        // exactly the sort of thing that goes unnoticed for a month.
        var sentences = new[]
        {
            "I'm sorry.", "I'd be glad to.", "Wow!", "The kettle is on.", "Is it?",
            "Unfortunately not.", "Of course.", "Nothing to report."
        };
        Check("every mood is a VRM preset",
            sentences.All(s => presets.Contains(Moods.Read(s).Name)),
            string.Join(",", sentences.Select(s => Moods.Read(s).Name).Where(m => !presets.Contains(m))));

        return failures;
    }
}
```
