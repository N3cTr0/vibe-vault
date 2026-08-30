---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\MusicChecks.cs
---

# tools\EarsTest\MusicChecks.cs

```csharp
// Beat detection is arithmetic, so it can be wrong quietly: she would simply dance a
// little off, or not at all, and nobody could say why. These feed the analyser material
// whose tempo is known by construction and check it agrees — and, just as importantly,
// check that speech is *not* mistaken for music, which is the failure that would put
// headphones on her every time somebody talked.
using System.Speech.AudioFormat;
using System.Speech.Synthesis;
using Octavia.Senses.Music;

internal static class MusicChecks
{
    private const int Rate = 48000;   // what a sound card's mix format almost always is

    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        // --- tempo, at three speeds --------------------------------
        foreach (var tempo in new[] { 90.0, 120.0, 150.0 })
        {
            var analyzer = new MusicAnalyzer(Rate);
            Feed(analyzer, Track(tempo, 20));
            var state = analyzer.State;

            Check($"{tempo:0} bpm track is heard as music", state.Playing,
                $"playing={state.Playing}, confidence {state.Confidence:0.00}, continuity {analyzer.Continuity:0.00}");

            // Half or double time is a genuine answer to "what is the period", so it is
            // worth knowing which one came back rather than only how far off it was.
            Check($"{tempo:0} bpm track measures its tempo", Math.Abs(state.Bpm - tempo) <= 6,
                $"measured {state.Bpm:0.0}");
        }

        // --- the beat clock ----------------------------------------
        {
            var analyzer = new MusicAnalyzer(Rate);
            var track = Track(120, 24);

            // The first stretch is warm-up; only beats after it are counted, because
            // there is nothing to be in time with until a tempo has been found.
            Feed(analyzer, track.AsSpan(0, Rate * 12).ToArray());
            var beats = Feed(analyzer, track.AsSpan(Rate * 12).ToArray());

            var expected = 12 * 120 / 60;
            Check("beats arrive at the tempo", Math.Abs(beats - expected) <= 2,
                $"{beats} beats in 12s, expected about {expected}");
        }

        // --- her own voice must not become music -------------------
        {
            var analyzer = new MusicAnalyzer(Rate);
            Feed(analyzer, Speech(20));
            var state = analyzer.State;

            Check("speech is not heard as music", !state.Playing,
                $"confidence {state.Confidence:0.00}, continuity {analyzer.Continuity:0.00}, {state.Bpm:0.0} bpm");
        }

        // --- silence -----------------------------------------------
        {
            var analyzer = new MusicAnalyzer(Rate);
            Feed(analyzer, new float[Rate * 10]);
            var state = analyzer.State;

            Check("silence is not heard as music", !state.Playing && state.Bpm == 0,
                $"playing={state.Playing}, {state.Bpm:0.0} bpm");
        }

        // --- talking over a track ----------------------------------
        // The design decision worth a test: she goes on hearing the beat while she
        // speaks, and her own voice cannot start or stop the music behind it.
        {
            var analyzer = new MusicAnalyzer(Rate);
            Feed(analyzer, Track(120, 16));
            var locked = analyzer.State;

            var beats = Feed(analyzer, Speech(6), hold: true);
            var after = analyzer.State;

            Check("she keeps the beat while speaking", beats >= 10,
                $"{beats} beats in 6s of held audio, expected about 12");
            Check("her own voice does not stop the music", after.Playing && locked.Playing,
                $"before={locked.Playing}, after={after.Playing}");
            Check("her own voice does not move the tempo", Math.Abs(after.Bpm - locked.Bpm) < 0.01,
                $"{locked.Bpm:0.0} became {after.Bpm:0.0}");
        }

        return failures;
    }

    /// Pushes audio in buffers the size the loopback actually delivers, so the analyser
    /// is exercised across buffer boundaries rather than handed one tidy array.
    private static int Feed(MusicAnalyzer analyzer, float[] samples, bool hold = false)
    {
        const int chunk = 480;   // ~10 ms, about what WASAPI hands over
        var beats = 0;

        for (var at = 0; at < samples.Length; at += chunk)
            beats += analyzer.Push(samples.AsSpan(at, Math.Min(chunk, samples.Length - at)), hold);

        return beats;
    }

    /// Something music-shaped: a sustained bass and pad so it does not stop between
    /// beats, with a percussive transient on each beat and a backbeat on two and four.
    /// A bare click track would be the easier test and the wrong one — real music is
    /// continuous, and continuity is one of the three things the analyser decides on.
    internal static float[] Track(double bpm, int seconds)
    {
        var samples = new float[Rate * seconds];
        var period = 60.0 / bpm * Rate;
        var random = new Random(1074);

        for (var i = 0; i < samples.Length; i++)
        {
            var t = i / (double)Rate;

            var bass = Math.Sin(2 * Math.PI * 80 * t) * 0.12;
            var pad = (Math.Sin(2 * Math.PI * 220 * t)
                     + Math.Sin(2 * Math.PI * 277 * t)
                     + Math.Sin(2 * Math.PI * 330 * t)) * 0.04;

            var beat = i / period;
            var since = (beat - Math.Floor(beat)) * period / Rate;   // seconds since the last beat
            var onBackbeat = ((int)Math.Floor(beat)) % 2 == 1;

            // The click matters as much as the thud. A kick modelled as a pure 55 Hz sine
            // occupies about one FFT bin and so barely shows up in spectral flux at all,
            // which would leave only the backbeat visible and make this a 75 bpm track
            // wearing a 150 bpm label. Every real kick drum has a beater attack.
            var kick = Math.Exp(-since * 26) * Math.Sin(2 * Math.PI * 55 * since) * 0.55
                     + Math.Exp(-since * 150) * (random.NextDouble() * 2 - 1) * 0.30;
            var snare = onBackbeat ? Math.Exp(-since * 34) * (random.NextDouble() * 2 - 1) * 0.35 : 0;

            samples[i] = (float)Math.Clamp(bass + pad + kick + snare, -1, 1);
        }

        return samples;
    }

    /// Synthesized speech with uneven gaps. The gaps are the point: they are what makes
    /// speech discontinuous, and the unevenness stops the repetition itself becoming a
    /// tempo the analyser could honestly find.
    private static float[] Speech(int seconds)
    {
        var clip = SpokenClip();
        var samples = new float[Rate * seconds];
        var random = new Random(9021);
        var at = 0;

        while (at < samples.Length)
        {
            var take = Math.Min(clip.Length, samples.Length - at);
            Array.Copy(clip, 0, samples, at, take);
            at += take + (int)(Rate * (0.35 + random.NextDouble() * 0.9));
        }

        return samples;
    }

    private static float[] SpokenClip()
    {
        var path = Path.Combine(Path.GetTempPath(), "octavia-musictest.wav");

        using (var synth = new SpeechSynthesizer())
        {
            synth.SetOutputToWaveFile(path,
                new SpeechAudioFormatInfo(Rate, AudioBitsPerSample.Sixteen, AudioChannel.Mono));
            synth.Speak("This is me talking, not music. There is no beat here to find at all.");
        }

        var bytes = File.ReadAllBytes(path);
        var samples = new float[(bytes.Length - 44) / 2];
        for (var i = 0; i < samples.Length; i++)
            samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;

        return samples;
    }
}
```
