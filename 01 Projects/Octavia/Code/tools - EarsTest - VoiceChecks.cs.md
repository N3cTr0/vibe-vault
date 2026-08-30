---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\VoiceChecks.cs
---

# tools\EarsTest\VoiceChecks.cs

```csharp
// The mouth is now read out of the waveform rather than handed to us by the engine, so
// it is our arithmetic that can be wrong. `EarsTest -- mouth <wav>` is for judging it by
// eye; these are the properties that must hold whatever it looks like.
using Octavia.Audio;
using Octavia.Voice;

internal static class VoiceChecks
{
    private const int Rate = 22050;

    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        var vocabulary = new[] { "aa", "ih", "ou", "ee", "oh" };

        // --- silence -------------------------------------------------
        var quiet = new VisemeReader(Rate);
        var silence = quiet.Read(new short[VisemeReader.FrameSamples]);
        Check("silence shuts her mouth", silence.Shape is null && silence.Openness == 0,
            $"{silence.Shape} {silence.Openness:0.00}");

        // A frame shorter than the window has nothing to say about anything.
        Check("a short frame is ignored", quiet.Read(new short[8]).Shape is null, "it returned a shape");

        // --- the two ends of the front/back axis ---------------------
        // Alternated rather than held: a constant spectrum pulls the adaptive centre
        // onto itself, which is correct behaviour and useless for telling them apart.
        var reader = new VisemeReader(Rate);
        var shapes = new List<string>();
        var openness = new List<double>();

        for (var round = 0; round < 40; round++)
        {
            var frame = Tone(round % 2 == 0 ? 320 : 2600, 0.30);
            var mouth = reader.Read(frame);
            if (mouth.Shape is not null) { shapes.Add(mouth.Shape); openness.Add(mouth.Openness); }
        }

        Check("tones produce shapes", shapes.Count > 20, $"{shapes.Count} voiced frames");
        Check("every shape is a VRM viseme", shapes.All(vocabulary.Contains),
            string.Join(",", shapes.Distinct().Except(vocabulary)));
        Check("low and high tones differ", shapes.Distinct().Count() > 1,
            $"everything read as '{shapes.FirstOrDefault()}'");

        var back = new[] { "oh", "ou" };
        Check("a low tone reads as a rounded mouth", shapes.Where((_, i) => i % 2 == 0).Any(back.Contains),
            string.Join(",", shapes.Where((_, i) => i % 2 == 0).Distinct()));
        Check("a high tone reads as a spread mouth", shapes.Where((_, i) => i % 2 == 1).Any(s => s is "ee" or "ih"),
            string.Join(",", shapes.Where((_, i) => i % 2 == 1).Distinct()));

        // --- loudness -------------------------------------------------
        var loudReader = new VisemeReader(Rate);
        for (var i = 0; i < 30; i++) loudReader.Read(Tone(500, 0.35));
        var loud = loudReader.Read(Tone(500, 0.35));
        var soft = loudReader.Read(Tone(500, 0.02));

        Check("loud opens further than soft", loud.Openness > soft.Openness,
            $"loud {loud.Openness:0.00} vs soft {soft.Openness:0.00}");
        Check("openness stays in range", loud.Openness is >= 0 and <= 1, $"{loud.Openness:0.00}");

        // --- the engine's own list ------------------------------------
        Check("piper names tidy up for a menu",
            PiperStore.Pretty("en_GB-jenny_dioco-medium") == "Jenny Dioco (en-GB, medium)",
            PiperStore.Pretty("en_GB-jenny_dioco-medium"));
        Check("an odd name survives tidying", PiperStore.Pretty("nonsense") == "nonsense",
            PiperStore.Pretty("nonsense"));
        Check("the catalogue is not empty", PiperStore.Catalogue.Count > 0, "no voices listed");

        return failures;
    }

    private static short[] Tone(double hz, double amplitude)
    {
        var frame = new short[VisemeReader.FrameSamples];
        for (var i = 0; i < frame.Length; i++)
            frame[i] = (short)(Math.Sin(2 * Math.PI * hz * i / Rate) * amplitude * short.MaxValue);
        return frame;
    }
}
```
