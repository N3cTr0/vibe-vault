---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\ComputeProbe.cs
---

# tools\EarsTest\ComputeProbe.cs

```csharp
using System.Diagnostics;
using System.Speech.AudioFormat;
using System.Speech.Synthesis;
using Octavia.Senses;

/// Diagnostic: how long a transcription takes on this machine's CPU versus its GPU.
///
/// Worth having because "auto" quietly means GPU — Whisper.net tries CUDA first — and
/// a weak card therefore wins over a strong CPU by default. That is a real regression
/// on some machines and a real speed-up on others, and the only way to know which is
/// to time it.
///
/// Whisper.net reads its runtime order once, when the native library first loads, so
/// one process can only measure one setting. This runs whichever it is told and prints
/// a line; run it twice and compare.
internal static class ComputeProbe
{
    public static async Task RunAsync(string compute, string model = "tiny.en", int threads = 0)
    {
        Console.WriteLine($"compute probe: {compute}, model {model}");

        var phrase = "Hello Octavia, can you hear me clearly today? "
                   + "This is a slightly longer sentence so the timing means something.";
        var wavPath = Path.Combine(Path.GetTempPath(), "octavia-computeprobe.wav");
        using (var synth = new SpeechSynthesizer())
        {
            synth.SetOutputToWaveFile(wavPath,
                new SpeechAudioFormatInfo(SileroVad.SampleRate, AudioBitsPerSample.Sixteen, AudioChannel.Mono));
            synth.Speak(phrase);
        }

        var bytes = File.ReadAllBytes(wavPath);
        var samples = new float[(bytes.Length - 44) / 2];
        for (var i = 0; i < samples.Length; i++)
            samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;

        var seconds = samples.Length / (float)SileroVad.SampleRate;
        Console.WriteLine($"audio: {seconds:0.0}s");

        var modelPath = await WhisperModelStore.EnsureAsync(model, m => Console.WriteLine($"  {m}"));

        using var whisper = new WhisperTranscriber(modelPath, "en", compute, threads);
        Console.WriteLine($"loaded library: {WhisperCompute.Loaded ?? "unreported"}");

        // The first pass carries model load and any kernel warm-up, which is not what
        // a conversation pays per utterance. Report it, then time three warm runs.
        var cold = Stopwatch.StartNew();
        var first = await whisper.TranscribeAsync(samples);
        cold.Stop();
        Console.WriteLine($"cold  {cold.Elapsed.TotalSeconds:0.00}s  \"{first.Text.Trim()}\"");

        var warm = new List<double>();
        for (var i = 0; i < 3; i++)
        {
            var sw = Stopwatch.StartNew();
            await whisper.TranscribeAsync(samples);
            sw.Stop();
            warm.Add(sw.Elapsed.TotalSeconds);
            Console.WriteLine($"warm  {sw.Elapsed.TotalSeconds:0.00}s");
        }

        warm.Sort();
        var median = warm[warm.Count / 2];
        Console.WriteLine();
        Console.WriteLine($"RESULT {compute}: median {median:0.00}s for {seconds:0.0}s of audio "
                        + $"({seconds / median:0.0}x realtime) on {WhisperCompute.Loaded ?? "unreported"}");
    }
}
```
