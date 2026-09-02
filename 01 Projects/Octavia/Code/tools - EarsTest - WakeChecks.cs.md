---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\WakeChecks.cs
---

# tools\EarsTest\WakeChecks.cs

```csharp
using System.Speech.AudioFormat;
using System.Speech.Synthesis;
using Octavia.Senses;

/// The wake word, driven with real speech through the real ONNX chain.
///
/// **Proved with `hey jarvis`, which is not her phrase and that is the point.** Hers has to
/// be trained off this machine, in a Colab, against 2022-era pinned dependencies — so if the
/// only way to know whether any of this worked were to train it first, nobody would find out
/// the plumbing was wrong until after ninety minutes of somebody else's GPU.
///
/// A pretrained model separates two questions that would otherwise arrive together: *does the
/// pipeline work* and *is this model any good*. The first is answered here, for nothing.
///
/// The speech is synthesised rather than recorded, which suits this unusually well: these
/// models are **trained on synthetic speech** in the first place.
internal static class WakeChecks
{
    private const string Phrase = "hey jarvis";

    public static async Task<int> RunAsync()
    {
        var failures = 0;
        void Check(string label, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        var model = WakeWordStore.ResolveWakeModel(Phrase);
        if (model is null) { Console.WriteLine("  skipped: no model name for the test phrase"); return 0; }

        if (!await WakeWordStore.EnsureAsync(model, m => Console.WriteLine($"  ..   {m}")))
        {
            // No network on this machine is not a failure of the wake word.
            Console.WriteLine("  skipped: the models could not be fetched");
            return 0;
        }

        using var wake = new WakeWord(
            Phrase,
            WakeWordStore.PathFor(WakeWordStore.Melspectrogram),
            WakeWordStore.PathFor(WakeWordStore.Embedding),
            WakeWordStore.PathFor(model));

        Check("the three models loaded", true, $"{Phrase}");

        /* Silence first, and before anything else is fed in.

           A wake word that fires on silence is worse than one that never fires: it would wake
           her at three in the morning, repeatedly, and every symptom would point at the
           microphone. */
        var quiet = Feed(wake, new float[16000 * 2], 0.5f);
        Check("silence does not wake her", !quiet, $"peak {wake.LastScore:0.000}");

        var spoken = Say(Phrase);
        Check("the phrase synthesised", spoken.Length > 8000, $"{spoken.Length / 16000.0:0.0}s");

        var woke = Feed(wake, spoken, 0.5f);
        Check($"'{Phrase}' wakes her", woke, $"peak {wake.LastScore:0.000}");

        /* Something else entirely, and deliberately a *request* rather than a nonsense
           phrase — "turn the kitchen light off" is exactly the kind of sentence she will
           hear all day without being addressed, and the one a false positive costs most. */
        var other = Say("turn the kitchen light off please");
        var wrongly = Feed(wake, other, 0.5f);
        Check("ordinary speech does not", !wrongly, $"peak {wake.LastScore:0.000}");

        // Twice in a row, because the window is cleared on a hit precisely so one utterance
        // cannot wake her repeatedly — and clearing it must not stop the *next* one working.
        var again = Feed(wake, Say(Phrase), 0.5f);
        Check("and it still works the second time", again, $"peak {wake.LastScore:0.000}");

        return failures;
    }

    /// Pushed in small chunks, the way a microphone delivers it — a single enormous call
    /// would exercise a path nothing in production takes.
    private static bool Feed(WakeWord wake, float[] samples, float threshold)
    {
        var fired = false;

        for (var at = 0; at < samples.Length; at += 1280)
        {
            var length = Math.Min(1280, samples.Length - at);
            if (wake.Heard(samples.AsSpan(at, length), threshold)) fired = true;
        }

        return fired;
    }

    /// Synthesised at exactly the rate the wake word expects, so nothing resamples.
    private static float[] Say(string text)
    {
        var path = Path.Combine(Path.GetTempPath(), $"octavia-wake-{Math.Abs(text.GetHashCode())}.wav");

        using (var synth = new SpeechSynthesizer())
        {
            synth.SetOutputToWaveFile(path,
                new SpeechAudioFormatInfo(16000, AudioBitsPerSample.Sixteen, AudioChannel.Mono));
            synth.Speak(text);
        }

        var bytes = File.ReadAllBytes(path);
        var samples = new float[(bytes.Length - 44) / 2];
        for (var i = 0; i < samples.Length; i++)
            samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;

        // A little silence in front, so the first window is not half-filled with the very
        // start of the word — a microphone always delivers some.
        var padded = new float[samples.Length + 8000];
        samples.CopyTo(padded, 8000);
        return padded;
    }
}
```
