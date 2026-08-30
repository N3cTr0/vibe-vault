---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\MouthProbe.cs
---

# tools\EarsTest\MouthProbe.cs

```csharp
using Octavia.Audio;

/// Diagnostic: what mouth the viseme reader makes of a piece of speech.
///
/// Lip sync is the one thing that cannot be judged from an assertion — it either looks
/// like talking or it does not. This prints the shape timeline for a WAV so the mapping
/// can be read, argued with and tuned against real audio.
internal static class MouthProbe
{
    public static void Run(string? path)
    {
        if (path is null || !File.Exists(path))
        {
            Console.WriteLine("usage: EarsTest -- mouth <file.wav>   (16-bit mono)");
            return;
        }

        var (samples, sampleRate) = ReadWav(path);
        Console.WriteLine($"{Path.GetFileName(path)}: {samples.Length / (double)sampleRate:0.00}s at {sampleRate} Hz");
        Console.WriteLine();

        var reader = new VisemeReader(sampleRate);
        var counts = new Dictionary<string, int>();
        var fronts = new List<double>();
        var frames = 0;
        var silent = 0;

        for (var offset = 0; offset + VisemeReader.FrameSamples <= samples.Length; offset += VisemeReader.FrameSamples)
        {
            var mouth = reader.Read(samples.AsSpan(offset, VisemeReader.FrameSamples));
            frames++;

            if (mouth.Shape is null) { silent++; }
            else
            {
                counts[mouth.Shape] = counts.GetValueOrDefault(mouth.Shape) + 1;
                fronts.Add(reader.LastFront);
            }

            // One line per 10 frames keeps a sentence readable at a glance.
            if (frames % 10 == 0 || mouth.Shape is not null)
            {
                var at = offset / (double)sampleRate;
                var bar = new string('#', (int)Math.Round(mouth.Openness * 30));
                Console.WriteLine($"{at,6:0.00}s  {mouth.Shape ?? "--",-3}  {mouth.Openness:0.00} {bar}");
            }
        }

        Console.WriteLine();
        Console.WriteLine($"frames {frames}, closed {silent} ({silent * 100.0 / Math.Max(frames, 1):0}%)");
        foreach (var (shape, count) in counts.OrderByDescending(p => p.Value))
            Console.WriteLine($"  {shape,-3} {count,4}  {count * 100.0 / Math.Max(frames - silent, 1):0}% of voiced");

        // Where the thresholds actually fall. Set them from this, not from intuition.
        if (fronts.Count > 0)
        {
            fronts.Sort();
            Console.WriteLine();
            Console.Write("front quantiles ");
            foreach (var q in new[] { 0.1, 0.25, 0.5, 0.75, 0.9 })
                Console.Write($" p{q * 100:0}={fronts[(int)(q * (fronts.Count - 1))]:0.000}");
            Console.WriteLine();
        }
    }

    /// Enough WAV parsing to walk the chunks and find the samples. Deliberately strict:
    /// a probe that silently misreads its input is worse than one that refuses.
    private static (short[] Samples, int SampleRate) ReadWav(string path)
    {
        var bytes = File.ReadAllBytes(path);
        if (bytes.Length < 44) throw new InvalidDataException("too short to be a WAV");

        var sampleRate = BitConverter.ToInt32(bytes, 24);
        int channels = BitConverter.ToInt16(bytes, 22);
        var bits = BitConverter.ToInt16(bytes, 34);
        if (bits != 16) throw new InvalidDataException($"{bits}-bit audio; this probe wants 16-bit");

        var offset = 12;
        while (offset + 8 <= bytes.Length)
        {
            var id = System.Text.Encoding.ASCII.GetString(bytes, offset, 4);
            var size = BitConverter.ToInt32(bytes, offset + 4);
            if (id == "data")
            {
                var start = offset + 8;
                var count = Math.Min(size, bytes.Length - start) / 2;
                var samples = new short[count / Math.Max(channels, 1)];

                // Mixed down, because the mouth only has one of everything.
                for (var i = 0; i < samples.Length; i++)
                {
                    var sum = 0;
                    for (var c = 0; c < channels; c++)
                        sum += BitConverter.ToInt16(bytes, start + (i * channels + c) * 2);
                    samples[i] = (short)(sum / channels);
                }

                return (samples, sampleRate);
            }

            offset += 8 + size + (size % 2);
        }

        throw new InvalidDataException("no data chunk");
    }
}
```
