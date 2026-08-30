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

        // --- SAPI viseme -> VRM mouth shape --------------------------
        var shapes = Enumerable.Range(0, 22).Select(SapiVoice.Shape).ToList();
        var vocabulary = new[] { "aa", "ih", "ou", "ee", "oh" };

        Check("silence closes the mouth", shapes[0] is null, $"got '{shapes[0]}'");
        Check("p/b/m closes the mouth", shapes[21] is null, $"got '{shapes[21]}'");
        Check("every shape is a VRM viseme",
            shapes.All(s => s is null || vocabulary.Contains(s)),
            string.Join(",", shapes.Where(s => s is not null && !vocabulary.Contains(s))));
        Check("all five shapes are reachable",
            vocabulary.All(v => shapes.Contains(v)),
            string.Join(",", vocabulary.Except(shapes.Where(s => s is not null)!)));
        Check("'aa' is an open vowel", SapiVoice.Shape(2) == "aa", SapiVoice.Shape(2) ?? "null");
        Check("'w/uw' rounds the lips", SapiVoice.Shape(7) == "ou", SapiVoice.Shape(7) ?? "null");
        Check("sibilants stay narrow", SapiVoice.Shape(15) == "ih", SapiVoice.Shape(15) ?? "null");

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
