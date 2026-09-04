---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\WakeProbe.cs
---

# tools\EarsTest\WakeProbe.cs

```csharp
using System.Speech.AudioFormat;
using System.Speech.Synthesis;
using Octavia.Senses;

/// What a trained wake word actually scores, rather than whether it passed.
///
/// **`WakeChecks` answers a different question and answers it well.** It proves the ONNX
/// chain works, with a pretrained model, against a fixed 0.5 — pass or fail, no numbers to
/// argue with. That is the right shape for a check that runs in the suite forever.
///
/// This is for the hour after a Colab finishes, when the question is not *does the plumbing
/// work* but *is this model any good, and where should the threshold sit*. openWakeWord's
/// 0.5 is generic guidance; the only threshold worth having is one read off the gap between
/// what this model scores on the phrase and what it scores on everything else.
///
/// **Several voices, because one voice measures one voice.** A model that fires for the
/// default SAPI speaker and nothing else would look perfect here and be useless in the room.
internal static class WakeProbe
{
    public static async Task RunAsync(string phrase)
    {
        var model = WakeWordStore.ResolveWakeModel(phrase);
        if (model is null)
        {
            Console.WriteLine($"no model for '{phrase}'.");
            Console.WriteLine($"expected {WakeWordStore.PathFor(phrase.Trim().ToLowerInvariant().Replace(' ', '_') + ".onnx")}");
            return;
        }

        if (!await WakeWordStore.EnsureAsync(model, m => Console.WriteLine($"  {m}"))) return;

        using var wake = new WakeWord(
            phrase,
            WakeWordStore.PathFor(WakeWordStore.Melspectrogram),
            WakeWordStore.PathFor(WakeWordStore.Embedding),
            WakeWordStore.PathFor(model));

        var size = new FileInfo(WakeWordStore.PathFor(model)).Length >> 10;
        Console.WriteLine($"model  : {model} ({size} KB)");

        var voices = Voices();
        Console.WriteLine($"voices : {string.Join(", ", voices)}");
        Console.WriteLine();

        var positives = new List<float>();
        var negatives = new List<float>();

        Console.WriteLine("  score   what");
        Console.WriteLine("  -----   ----");

        // Silence first. A wake word that scores on nothing at all is not worth measuring
        // further, and it is the one failure that would wake her at three in the morning.
        Report(wake, "(silence)", new float[16000 * 2], negatives);

        foreach (var voice in voices)
        {
            Report(wake, $"\"{phrase}\"  [{voice}]", Say(phrase, voice), positives);
        }

        /* Ordinary sentences, deliberately the kind she will hear all day without being
           addressed — a false positive on one of these costs a Whisper transcription and a
           turn she was never asked for. */
        foreach (var line in new[]
        {
            "turn the kitchen light off please",
            "I think it is going to rain later",
            "did you see what happened in the second half"
        })
        {
            Report(wake, $"\"{line}\"", Say(line, voices[0]), negatives);
        }

        /* Near misses, which are the interesting negatives. Her name on its own is what
           somebody says when they are talking *about* her, and the rest are the acoustic
           neighbours a two-word phrase actually has. */
        foreach (var line in new[] { "octavia", "hey, are you there", "hey octopus" })
        {
            Report(wake, $"\"{line}\"  (near miss)", Say(line, voices[0]), negatives);
        }

        Console.WriteLine();
        Summarise(positives, negatives);
    }

    private static void Summarise(List<float> positives, List<float> negatives)
    {
        if (positives.Count == 0) { Console.WriteLine("nothing to summarise."); return; }

        var weakest = positives.Min();
        var loudest = negatives.Count == 0 ? 0f : negatives.Max();

        Console.WriteLine($"weakest hit on the phrase : {weakest:0.000}");
        Console.WriteLine($"loudest score on anything else : {loudest:0.000}");

        if (weakest <= loudest)
        {
            Console.WriteLine();
            Console.WriteLine("THERE IS NO THRESHOLD THAT WORKS. Something it should ignore scores");
            Console.WriteLine("at least as high as something it should catch, so any setting either");
            Console.WriteLine("misses her name or fires on ordinary speech. This needs more training");
            Console.WriteLine("examples, not a different number.");
            return;
        }

        // Midway on a log-ish scale would be fiddlier than it is worth; the gap is what
        // matters, and a wide one makes the exact pick uninteresting.
        var suggested = (weakest + loudest) / 2f;
        Console.WriteLine($"gap : {weakest - loudest:0.000}");
        Console.WriteLine();
        Console.WriteLine($"WakeThreshold {suggested:0.00} sits in the middle of that gap.");
        Console.WriteLine("Measured against synthesised speech, so treat it as a starting point and");
        Console.WriteLine("read the real answer off the declined-utterance scores in her log.");
    }

    private static void Report(WakeWord wake, string label, float[] samples, List<float> into)
    {
        var peak = Peak(wake, samples);
        into.Add(peak);
        Console.WriteLine($"  {peak:0.000}   {label}");
    }

    /// The highest score anywhere in the clip. Fed at a threshold nothing can reach, so the
    /// window is never cleared and one utterance is measured as one utterance.
    private static float Peak(WakeWord wake, float[] samples)
    {
        var peak = 0f;

        for (var at = 0; at < samples.Length; at += 1280)
        {
            var length = Math.Min(1280, samples.Length - at);
            wake.Heard(samples.AsSpan(at, length), 2f);
            peak = Math.Max(peak, wake.LastScore);
        }

        // Trailing silence, so the phrase is pushed all the way through the sliding window
        // rather than being measured while it is still half outside it.
        var tail = new float[16000];
        for (var at = 0; at < tail.Length; at += 1280)
        {
            wake.Heard(tail.AsSpan(at, Math.Min(1280, tail.Length - at)), 2f);
            peak = Math.Max(peak, wake.LastScore);
        }

        return peak;
    }

    private static string[] Voices()
    {
        using var synth = new SpeechSynthesizer();
        var installed = synth.GetInstalledVoices()
            .Where(v => v.Enabled)
            .Select(v => v.VoiceInfo.Name)
            .ToArray();

        return installed.Length == 0 ? [""] : installed.Take(3).ToArray();
    }

    private static float[] Say(string text, string voice)
    {
        var path = Path.Combine(Path.GetTempPath(),
            $"octavia-probe-{Math.Abs((text + voice).GetHashCode())}.wav");

        using (var synth = new SpeechSynthesizer())
        {
            if (voice.Length > 0) { try { synth.SelectVoice(voice); } catch { /* the default will do */ } }
            synth.SetOutputToWaveFile(path,
                new SpeechAudioFormatInfo(16000, AudioBitsPerSample.Sixteen, AudioChannel.Mono));
            synth.Speak(text);
        }

        var bytes = File.ReadAllBytes(path);
        var samples = new float[(bytes.Length - 44) / 2];
        for (var i = 0; i < samples.Length; i++)
            samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;

        var padded = new float[samples.Length + 8000];
        samples.CopyTo(padded, 8000);
        return padded;
    }
}
```
