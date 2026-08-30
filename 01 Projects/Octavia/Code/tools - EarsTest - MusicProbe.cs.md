---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\MusicProbe.cs
---

# tools\EarsTest\MusicProbe.cs

```csharp
using NAudio.Wave;
using Octavia.Senses.Music;

/// Diagnostic: what she makes of whatever this machine is playing right now.
///
/// A tempo cannot be judged from an assertion any more than lip sync can — put a track
/// on, run this, and see whether the number it settles on is the number you would tap.
/// It also answers the duller question first: whether there is an output device to
/// listen to at all, which over Remote Desktop there very often is not.
internal static class MusicProbe
{
    public static async Task RunAsync(bool demo = false)
    {
        Console.WriteLine($"default output: {LoopbackListener.DefaultDevice() ?? "NONE"}");

        using var watcher = new MusicWatcher();
        if (!await watcher.StartAsync())
        {
            Console.WriteLine("could not open the loopback capture; nothing to listen to.");
            return;
        }

        var beats = 0;
        watcher.Beat += () => beats++;

        // `demo` plays a track of a known tempo out of the speakers and listens to it
        // coming back, which is the only way to prove the whole chain on a machine
        // rather than the arithmetic in the middle of it. 132 is deliberately not a
        // round number and not one the checks use.
        using var output = demo ? Play(132) : null;

        Console.WriteLine(demo
            ? $"listening to '{watcher.DeviceName}' while playing a 132 bpm track through it."
            : $"listening to '{watcher.DeviceName}'. Play something. Ctrl+C to stop.");
        Console.WriteLine();

        var started = DateTime.UtcNow;
        while ((DateTime.UtcNow - started).TotalSeconds < (demo ? 30 : 60))
        {
            Thread.Sleep(500);
            var state = watcher.State;

            Console.WriteLine(
                $"{(DateTime.UtcNow - started).TotalSeconds,5:0.0}s  " +
                $"{(state.Playing ? "MUSIC" : "  -  ")}  " +
                $"{state.Bpm,5:0.0} bpm  " +
                $"energy {state.Energy:0.00}  " +
                $"confidence {state.Confidence:0.00}  " +
                $"beats {beats}");
        }

        // Whether the machine actually handed over every sample it played. A shortfall
        // here means the tempo was never findable, however good the arithmetic is.
        var elapsed = (DateTime.UtcNow - started).TotalSeconds;
        var (frames, gaps, silent, crest) = watcher.Delivery;
        var expected = watcher.SampleRate * elapsed;

        Console.WriteLine();
        Console.WriteLine($"delivery: {frames:n0} frames in {elapsed:0.0}s at {watcher.SampleRate} Hz " +
                          $"— {frames / expected:P1} of what that rate implies, " +
                          $"{gaps} discontinuities, {silent} silent buffers");
        Console.WriteLine($"dynamics: crest factor {crest:0.0}");

        if (crest is > 0 and < 2.5)
            Console.WriteLine(
                "          That is flat enough to be a limiter, not music. Something in the\n" +
                "          audio path is normalising everything to full scale, which leaves no\n" +
                "          transients for a beat to be found in. Remote Desktop's \"Remote Audio\"\n" +
                "          endpoint does this at any volume. Try it on the machine's own sound card.");
    }

    /// Pushes a generated track out of the default output, on a loop, so there is
    /// something real for the loopback to hear.
    private static WaveOut Play(double bpm)
    {
        var samples = MusicChecks.Track(bpm, 30);
        var pcm = new byte[samples.Length * 2];

        for (var i = 0; i < samples.Length; i++)
        {
            var value = (short)(Math.Clamp(samples[i], -1f, 1f) * 26000);
            pcm[i * 2] = (byte)(value & 0xFF);
            pcm[i * 2 + 1] = (byte)(value >> 8);
        }

        var buffer = new BufferedWaveProvider(new WaveFormat(48000, 16, 1), TimeSpan.FromSeconds(35))
        {
            DiscardOnBufferOverflow = true
        };
        buffer.AddSamples(pcm, 0, pcm.Length);

        var output = new WaveOut { BufferMilliseconds = 80, NumberOfBuffers = 3 };
        output.Init(buffer);
        output.Play();
        return output;
    }
}
```
