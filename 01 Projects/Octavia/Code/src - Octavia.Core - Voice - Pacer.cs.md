---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\Pacer.cs
---

# src\Octavia.Core\Voice\Pacer.cs

```csharp
using NAudio.Wave;

namespace Octavia.Voice;

/// The clock her voice used to get from a sound card.
///
/// **A server holds no output device**, and the thing that made that hard was never the
/// loudspeaker — it was that `WaveOut` pulled from the buffer in real time, and the audio
/// teed to faces, the visemes and the end-of-utterance were all consequences of that pull.
/// Take the device away and her mouth stops moving; take it away carelessly and her voice is
/// generated as fast as the engine can write it, arriving at a face seconds before it should.
///
/// So this pulls at the same rate, from the same buffer, into the same tap — and nothing it
/// reads goes anywhere near a device.
///
/// It lives in its own file because it outlived the engine it was written for: Piper was
/// replaced by Kokoro in Stage 16 and not one line of this changed, which is the strongest
/// evidence available that the clock was never part of the engine.
internal sealed class Pacer : IDisposable
{
    /// Small enough that a viseme lands close to the sound it belongs to; large enough that
    /// the thread is asleep almost all the time. `WaveOut` was using 80 ms in three buffers,
    /// so this is if anything tighter than what it replaces.
    private const int ChunkMs = 20;

    private readonly CancellationTokenSource _stopping = new();
    private readonly Thread _thread;

    public Pacer(IWaveProvider source, Action<ReadOnlySpan<byte>> tap)
    {
        var format = source.WaveFormat;
        var chunk = format.AverageBytesPerSecond * ChunkMs / 1000;

        // Whole samples only: half of a 16-bit sample read as the start of the next one is
        // a burst of noise and a mouth that opens at the wrong moment.
        chunk -= chunk % format.BlockAlign;

        _thread = new Thread(() => Run(source, tap, chunk))
        {
            IsBackground = true,
            Name = "octavia-voice-clock",

            // The one thing a sound card gave for free. A late pull is heard as a gap, and
            // this thread does almost nothing, so it can afford to be woken promptly.
            Priority = ThreadPriority.AboveNormal
        };

        _thread.Start();
    }

    private void Run(IWaveProvider source, Action<ReadOnlySpan<byte>> tap, int chunk)
    {
        var buffer = new byte[chunk];
        var started = System.Diagnostics.Stopwatch.StartNew();
        var due = TimeSpan.Zero;

        /* Paced against a stopwatch rather than by sleeping for the chunk length.

           Sleeping 20 ms in a loop drifts, because every sleep is *at least* 20 ms and the
           work in between is never free — a minute of speech would arrive noticeably late,
           and the visemes with it. Counting from a fixed start instead means an overrun is
           absorbed by the next wait rather than added to it. */
        while (!_stopping.IsCancellationRequested)
        {
            due += TimeSpan.FromMilliseconds(ChunkMs);

            var wait = due - started.Elapsed;
            if (wait > TimeSpan.Zero) _stopping.Token.WaitHandle.WaitOne(wait);
            if (_stopping.IsCancellationRequested) break;

            // Behind by more than a moment — the machine was busy, or she has been idle for
            // a long time. Catching up would emit a burst of audio at once; the clock resets
            // to now instead, which costs nothing because silence is what was missed.
            if (started.Elapsed - due > TimeSpan.FromMilliseconds(500)) due = started.Elapsed;

            int read;
            try { read = source.Read(buffer.AsSpan()); }
            catch (Exception ex) { Core.Log.Warn($"voice clock: {ex.Message}"); break; }

            if (read <= 0) continue;

            /* `BufferedWaveProvider` pads with silence when it has nothing, so a read that
               succeeds is not the same as audio existing. Teeing that padding would send
               every face a permanent stream of silence and drive her mouth from it. */
            if (source is BufferedWaveProvider { BufferedBytes: 0 }) continue;

            try { tap(buffer.AsSpan(0, read)); }
            catch (Exception ex) { Core.Log.Warn($"voice clock tap: {ex.Message}"); }
        }
    }

    public void Dispose()
    {
        _stopping.Cancel();
        _thread.Join(TimeSpan.FromMilliseconds(500));
        _stopping.Dispose();
    }
}
```
