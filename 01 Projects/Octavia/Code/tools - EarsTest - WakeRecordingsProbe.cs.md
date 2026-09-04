---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\WakeRecordingsProbe.cs
---

# tools\EarsTest\WakeRecordingsProbe.cs

```csharp
using Octavia.Senses;

/// Scores a wake word against a folder of real recordings.
///
/// **Every measurement of this model until now has been synthetic speech judged by a model
/// trained on synthetic speech.** `wakescore` uses SAPI voices; the training benchmark uses
/// piper clips. Both said the model was fine while the owner's ordinary voice was not being
/// heard, which is the only result that mattered.
///
/// Fifty real clips give a distribution rather than an anecdote: not *"it fired that time"*
/// but how many of fifty clear it, and by how much. That is the number a threshold should be
/// chosen from, and the number a retrain has to beat.
internal static class WakeRecordingsProbe
{
    public static async Task<int> RunAsync(string phrase, string folder, float threshold)
    {
        if (!Directory.Exists(folder)) { Console.WriteLine($"no folder: {folder}"); return 1; }

        var files = Directory.GetFiles(folder, "*.wav").OrderBy(f => f).ToArray();
        if (files.Length == 0) { Console.WriteLine($"no .wav files in {folder}"); return 1; }

        var model = WakeWordStore.ResolveWakeModel(phrase);
        if (model is null) { Console.WriteLine($"no model for '{phrase}'"); return 1; }
        if (!await WakeWordStore.EnsureAsync(model, Console.WriteLine)) return 1;

        using var wake = new WakeWord(
            phrase,
            WakeWordStore.PathFor(WakeWordStore.Melspectrogram),
            WakeWordStore.PathFor(WakeWordStore.Embedding),
            WakeWordStore.PathFor(model));

        Console.WriteLine($"model     : {model}");
        Console.WriteLine($"recordings: {files.Length} in {folder}");
        Console.WriteLine($"threshold : {threshold:0.00}");
        Console.WriteLine();

        var scores = new List<float>();
        foreach (var file in files)
        {
            // Per file, because *which* clips fail is the finding: if the misses are the
            // deliberately awkward ones -- muttered, turned away, far off -- that is a very
            // different verdict from a model that misses ordinary speech.
            var peak = Peak(wake, Read(file));
            scores.Add(peak);
            Console.WriteLine($"  {peak:0.000}  {Path.GetFileNameWithoutExtension(file)}");
        }

        var fired = scores.Count(s => s >= threshold);
        var ordered = scores.OrderBy(s => s).ToArray();

        // A histogram, because the shape is the finding. A model that is merely weak produces
        // a spread; one that is deaf to a voice produces a heap at zero and a few outliers,
        // and only the second is beyond rescue by a threshold.
        Console.WriteLine("  score      clips");
        foreach (var (low, high) in new[]
                 { (0.00f, 0.05f), (0.05f, 0.10f), (0.10f, 0.20f), (0.20f, 0.30f),
                   (0.30f, 0.50f), (0.50f, 0.70f), (0.70f, 1.01f) })
        {
            var n = scores.Count(s => s >= low && s < high);
            Console.WriteLine($"  {low:0.00}-{high:0.00}  {new string('#', n)} {(n > 0 ? n.ToString() : "")}");
        }

        Console.WriteLine();
        Console.WriteLine($"  fired at {threshold:0.00} : {fired}/{files.Length}  ({fired * 100.0 / files.Length:0}%)");
        Console.WriteLine($"  median            : {ordered[ordered.Length / 2]:0.000}");
        Console.WriteLine($"  best / worst      : {ordered[^1]:0.000} / {ordered[0]:0.000}");
        Console.WriteLine();

        // What the threshold could buy, if the scores are there to buy it with.
        foreach (var t in new[] { 0.05f, 0.10f, 0.15f, 0.20f, 0.30f })
            Console.WriteLine($"  at {t:0.00} it would fire on {scores.Count(s => s >= t)}/{files.Length}");

        return 0;
    }

    /// The highest score anywhere in the clip, fed at a threshold nothing reaches so the
    /// classifier's window is never cleared mid-clip.
    private static float Peak(WakeWord wake, float[] samples)
    {
        var peak = 0f;
        var padded = new float[samples.Length + 16000];   // trailing silence, to push it through
        samples.CopyTo(padded, 0);

        for (var at = 0; at < padded.Length; at += 1280)
        {
            wake.Heard(padded.AsSpan(at, Math.Min(1280, padded.Length - at)), 2f);
            peak = Math.Max(peak, wake.LastScore);
        }

        return peak;
    }

    /// 16 kHz mono 16-bit, which is what the recorder writes and what the models expect.
    private static float[] Read(string path)
    {
        var bytes = File.ReadAllBytes(path);
        var samples = new float[(bytes.Length - 44) / 2];
        for (var i = 0; i < samples.Length; i++)
            samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;
        return samples;
    }
}
```
