---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\SourceChecks.cs
---

# tools\EarsTest\SourceChecks.cs

```csharp
// The audio-source seam from Stage 14 item 2, and the two traps that come with it.
//
// The traps are the point. Swapping where her ears listen is easy; the ways it goes wrong
// are (a) her sense of what is playing *in this room* silently following the phone, and
// (b) the deaf-microphone warning firing at somebody who is simply not holding a button.
// Both would look like something else entirely, so both are checked here.
using Octavia.Senses;

internal static class SourceChecks
{
    public static int Run(string modelPath)
    {
        var failures = 0;

        void Check(string name, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        // --- framing -----------------------------------------------------------------
        var framer = new PcmFramer();
        var frames = 0;
        var wrongSize = 0;
        var firstSample = 0f;

        framer.Frame += (frame, count) =>
        {
            if (count != SileroVad.FrameSamples || frame.Length != SileroVad.FrameSamples) wrongSize++;
            if (frames == 0) firstSample = frame[0];
            frames++;
        };

        // Three frames' worth of bytes, plus a straggler that must be carried, not dropped.
        var bytes = new byte[SileroVad.FrameSamples * 2 * 3 + 10];
        bytes[0] = 0x00; bytes[1] = 0x40;     // 0x4000 = 16384 -> 0.5
        framer.Push(bytes, bytes.Length);

        Check("framing produces whole frames", frames == 3, $"{frames} frames");
        Check("every frame is the size the VAD wants", wrongSize == 0, $"{wrongSize} wrong");
        Check("little-endian samples are decoded", Math.Abs(firstSample - 0.5f) < 0.001f, $"{firstSample}");

        // A partial frame must be held for the next push rather than emitted short.
        framer.Push(new byte[SileroVad.FrameSamples * 2 - 10], SileroVad.FrameSamples * 2 - 10);
        Check("a straggler is carried into the next frame", frames == 4, $"{frames} frames");

        // --- the sources -------------------------------------------------------------
        using var face = new FaceAudioSource("a test face");
        var got = 0;
        face.Data += (_, n) => got += n;

        face.Push(new byte[64], 64);
        Check("a stopped face source passes nothing", got == 0, $"{got} bytes");

        face.Start();
        face.Push(new byte[64], 64);
        Check("a started face source passes frames", got == 64, $"{got} bytes");

        face.Stop();
        face.Push(new byte[64], 64);
        Check("stopping closes the tap again", got == 64, $"{got} bytes");

        // The counterpart to this asserted that a `LocalMicSource` *was* continuous. There is
        // no local microphone any more — Stage 15 item 3 — and `SpySource` below still covers
        // both sides of the flag, which is what the distinction is actually for.
        Check("a face is not expected to be continuous", !face.ExpectsContinuousAudio);

        /* --- trap 1, as a test ------------------------------------------------------
           Handing the ears a face **must not stop the source they were using**. The local
           microphone is shared with the room-music analyser, so stopping it here would
           silence her sense of this room for exactly as long as somebody held the floor
           somewhere else — and everything would still appear to work. */
        using var ears = new WhisperRecognizer(modelPath, "tiny.en", "en");
        var first = new SpySource("first", continuous: true);
        var second = new SpySource("second", continuous: false);

        ears.UseSource(first);
        ears.Start();
        ears.UseSource(second);

        Check("switching source leaves the old one running", !first.Stopped,
            "the shared local microphone was stopped, which would silence room music");
        Check("switching source attaches the new one", second.Started);

        // And the old source must no longer reach the recogniser, or she would transcribe
        // both rooms at once.
        Check("the old source is detached", first.Listeners == 0, $"{first.Listeners} still attached");

        ears.Stop();
        return failures;
    }

    /// A source that records what was done to it. No hardware, no audio.
    private sealed class SpySource(string name, bool continuous) : IAudioSource
    {
        public event Action<byte[], int>? Data;

        public string Name { get; } = name;
        public bool ExpectsContinuousAudio { get; } = continuous;

        public bool Started { get; private set; }
        public bool Stopped { get; private set; }
        public int Listeners => Data?.GetInvocationList().Length ?? 0;

        public void Start() => Started = true;
        public void Stop() => Stopped = true;
        public void Dispose() { }
    }
}
```
