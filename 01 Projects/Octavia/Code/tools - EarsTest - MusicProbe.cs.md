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
    public static async Task RunAsync(bool demo = false, string? device = null)
    {
        Console.WriteLine($"default output: {LoopbackListener.DefaultDevice() ?? "NONE"}");

        using var watcher = new MusicWatcher { Device = device };
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
        using var output = demo ? Play(132, device) : null;

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
        var (capturedPeak, capturedRms) = watcher.Levels;
        Console.WriteLine($"dynamics: crest factor {crest:0.0}  (peak {capturedPeak:0.000}, rms {capturedRms:0.000})");

        /* A captured crest factor means nothing on its own — it has to be compared with
           what was played. The demo track is synthetic and deliberately dense, so its
           own crest is far below real music's; judging the audio path against an
           absolute threshold therefore accuses the machine of limiting when the test
           signal was flat to begin with. That mistake was made here: the same ~1.7 was
           read over Remote Desktop, through a virtual streaming endpoint and through a
           real headset, and blamed on all three in turn. */
        if (demo)
        {
            var source = Crest(MusicChecks.Track(132, 30));
            Console.WriteLine($"          the track played has crest {source:0.0}, so anything near that is faithful capture");

            if (crest > source * 0.8)
            {
                Console.WriteLine("          CAPTURE IS FAITHFUL — the path preserved the dynamics it was given.");
                return;
            }
        }

        if (crest is > 0 and < 2.5)
            Console.WriteLine(
                "          That is flat enough to be a limiter, not music. Something in the\n" +
                "          audio path is normalising everything to full scale, which leaves no\n" +
                "          transients for a beat to be found in. Remote Desktop's \"Remote Audio\"\n" +
                "          endpoint does this at any volume. Try it on the machine's own sound card.");
    }

    /// Peak over RMS, the same arithmetic LoopbackListener reports for what it captured.
    private static double Crest(float[] samples)
    {
        double peak = 0, sum = 0;
        foreach (var sample in samples)
        {
            var magnitude = Math.Abs(sample);
            if (magnitude > peak) peak = magnitude;
            sum += (double)sample * sample;
        }

        var rms = Math.Sqrt(sum / Math.Max(1, samples.Length));
        return rms > 0 ? peak / rms : 0;
    }

    /// Pushes a generated track out of an output, so there is something of a known
    /// tempo and a known crest factor for the loopback to hear.
    ///
    /// It must play to the *same* endpoint the capture is tapping, or the test compares
    /// one device's dynamics with another's and the result means nothing.
    private static WaveOut Play(double bpm, string? device)
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

        // WaveOut names are truncated to 31 characters like WaveIn's, so match either
        // way round rather than comparing whole names.
        if (!string.IsNullOrWhiteSpace(device))
        {
            for (var i = 0; i < WaveOut.DeviceCount; i++)
            {
                var name = WaveOut.GetCapabilities(i).ProductName;
                if (name.Contains(device, StringComparison.OrdinalIgnoreCase) ||
                    device.Contains(name, StringComparison.OrdinalIgnoreCase))
                {
                    output.DeviceNumber = i;
                    Console.WriteLine($"playing through '{name}'");
                    break;
                }
            }
        }

        output.Init(buffer);
        output.Play();
        return output;
    }
}
```
