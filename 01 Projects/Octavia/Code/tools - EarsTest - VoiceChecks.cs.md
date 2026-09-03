---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\VoiceChecks.cs
---

# tools\EarsTest\VoiceChecks.cs

```csharp
using System.Runtime.InteropServices;
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

        /* --- the one voice --------------------------------------------

           `Speaker` is an index into `voices.bin` and **belongs to the model, not to us** —
           the same class of constant as openWakeWord's `/10 + 2`. Nothing enforces it,
           nothing errors, and a wrong one produces a perfectly good voice belonging to
           somebody else. Written down here so a change has to be deliberate. */
        Check("her voice is the one that was chosen", KokoroStore.Voice == "af_heart", KokoroStore.Voice);
        Check("and it is that voice's own index", KokoroStore.Speaker == 3, $"{KokoroStore.Speaker}");

        // A viseme frame is 512 samples; her clock delivers 20 ms. If a rate ever made
        // 20 ms *bigger* than a frame, the carry above would stop being exercised at all
        // and the bug it exists for could come back unnoticed.
        Check("her rate keeps a clock tick smaller than a viseme frame",
            KokoroStore.SampleRate * 20 / 1000 < VisemeReader.FrameSamples,
            $"{KokoroStore.SampleRate * 20 / 1000} samples per tick vs {VisemeReader.FrameSamples}");

        // The engine is shipped, not downloaded — so a missing one is a broken install, and
        // saying which is the difference between waiting and going to look for a file.
        Check("her voice engine is installed", KokoroStore.EnginePath() is not null,
            "octavia-kokoro.exe is not beside her or in a working tree; README.md publishes three projects");

        /* **The chunk size must not be able to silence her mouth** — v0.39.1.

           `VisemeReader` consumes fixed 512-sample frames, and `OnAudioPlayed` used to read
           whole frames out of whatever arrived and discard the remainder. That was safe only
           because a sound card delivered 80 ms at a time, or 1,764 samples. Replacing it with
           a 20 ms clock made every chunk **441 samples** — less than one frame — so the loop
           never ran and her mouth stopped moving entirely, while her voice played perfectly
           and nothing anywhere threw.

           So this asserts the property rather than the number: **audio delivered in pieces
           smaller than a frame still produces visemes.** Anyone changing the pacing again
           gets told here rather than by a bug report. */
        var chunked = new VisemeReader(Rate);
        var speech = new short[Rate];             // one second of a vowel-ish tone
        for (var i = 0; i < speech.Length; i++)
            speech[i] = (short)(12000 * Math.Sin(2 * Math.PI * 700 * i / (double)Rate));

        int Visemes(int chunk)
        {
            var seen = 0;
            var carry = new List<short>();

            for (var at = 0; at < speech.Length; at += chunk)
            {
                var length = Math.Min(chunk, speech.Length - at);
                carry.AddRange(speech.AsSpan(at, length).ToArray());

                while (carry.Count >= VisemeReader.FrameSamples)
                {
                    if (chunked.Read(CollectionsMarshal.AsSpan(carry)[..VisemeReader.FrameSamples]).Shape is not null)
                        seen++;
                    carry.RemoveRange(0, VisemeReader.FrameSamples);
                }
            }

            return seen;
        }

        var big = Visemes(1764);      // what a sound card used to deliver
        var small = Visemes(441);     // what the 20 ms clock delivers

        Check("a sound-card-sized chunk produces visemes", big > 0, $"{big}");
        Check("and a chunk smaller than one frame still does", small > 0,
            $"{small} — carrying leftovers across chunks is what makes this work");
        Check("...and produces about as many", Math.Abs(big - small) <= 2, $"{big} vs {small}");

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
