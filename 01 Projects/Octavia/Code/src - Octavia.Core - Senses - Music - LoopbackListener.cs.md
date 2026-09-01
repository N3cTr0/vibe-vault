---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Music\LoopbackListener.cs
---

# src\Octavia.Core\Senses\Music\LoopbackListener.cs

```csharp
using NAudio.CoreAudioApi;
using NAudio.Wave;
using Octavia.Core;

namespace Octavia.Senses.Music;

/// What the machine is playing, as mono samples.
///
/// WASAPI loopback taps the render endpoint itself, so she hears the output mix without
/// a cable, a virtual device, or anything routed through her. This is the capability
/// that stopped her being a browser page: no web API can hear the system.
///
/// **Nothing is recorded and nothing leaves.** Buffers are converted, handed to the
/// analyser, and dropped; what survives is a tempo and a loudness. See PROTOCOL.md.
internal sealed class LoopbackListener : IDisposable
{
    private WasapiRecorder? _recorder;
    private float[] _mono = new float[4096];

    /// Mono samples at `SampleRate`, on a capture thread.
    public event Action<float[], int>? Frames;

    /// The capture ended on its own — almost always because the output device changed
    /// under it. Whoever started it decides whether to open the new one.
    public event Action? Stopped;

    public int SampleRate { get; private set; }

    /// Which render endpoint to tap. Empty follows the Windows default.
    public string? Device { get; set; }
    public string DeviceName { get; private set; } = "none";
    public bool IsRunning => _recorder is not null;

    /// Delivery statistics. A tempo that will not settle is usually the audio path
    /// dropping or re-timing buffers rather than the arithmetic being wrong, and these
    /// separate the two: compare `FramesSeen` against the sample rate times the elapsed
    /// time, and watch whether `Gaps` is climbing.
    public long FramesSeen { get; private set; }
    public long Gaps { get; private set; }
    public long SilentBuffers { get; private set; }

    /// Peak over root-mean-square for everything heard so far — the crest factor.
    ///
    /// Music runs about 4 to 10. A path that is limiting or compressing pins it near 1.5,
    /// because everything arrives at full scale, and a beat cannot be found in audio with
    /// no dynamics left in it.
    ///
    /// **A reading near 1.73 with an RMS near 0.577 is not a limiter — it is this class
    /// misreading the samples.** Those are uniform noise's figures, and they were what
    /// `Sample` produced for years by taking the low two bytes of a 32-bit endpoint. The
    /// blame went to Remote Desktop, then to a virtual streaming endpoint, then to a
    /// headset, before anyone compared the number against the track being played. Compare
    /// against the source before concluding anything: `EarsTest music demo` now does.
    public double Crest
    {
        get
        {
            var rms = _frames > 0 ? Math.Sqrt(_energySum / _frames) : 0;
            return rms > 1e-9 ? _peak / rms : 0;
        }
    }

    /// The two halves of Crest, separately. A crest factor alone cannot distinguish
    /// "the peaks were squashed" from "the whole signal arrived far too quiet", and
    /// those need very different answers.
    public double Peak => _peak;
    public double Rms => _frames > 0 ? Math.Sqrt(_energySum / _frames) : 0;

    private double _peak, _energySum;
    private long _frames;

    /// The endpoint she would listen to, without opening it. The self-test needs to be
    /// able to say "there is no output device" without starting a capture to find out.
    public static string? DefaultDevice()
    {
        try
        {
            using var enumerator = new MMDeviceEnumerator();
            if (!enumerator.TryGetDefaultAudioEndpoint(DataFlow.Render, Role.Multimedia, out var device))
                return null;

            using (device) return device.FriendlyName;
        }
        catch
        {
            return null;
        }
    }

    /// Asynchronous because stream routing is: the recorder has to negotiate with the
    /// endpoint before it can promise to follow the default device around.
    public async Task<bool> StartAsync()
    {
        if (_recorder is not null) return true;

        try
        {
            // Choosing the endpoint matters more here than anywhere else she listens:
            // the Windows default on a machine with streaming software is often a
            // virtual device that normalises to full scale, and a beat cannot be found
            // in audio with no dynamics left in it.
            var device = AudioDevices.Resolve(DataFlow.Render, Device);
            if (device is null)
            {
                Log.Warn(string.IsNullOrWhiteSpace(Device)
                    ? "music: no audio output device to listen to"
                    : $"music: no output device matching '{Device}'");
                return false;
            }

            // The endpoint is pinned, because automatic stream routing follows the
            // default *capture* device and cannot be combined with loopback. Headphones
            // going in therefore ends this capture rather than moving it, which is what
            // `Stopped` below exists to notice.
            using (device)
            {
                var recorder = await new WasapiRecorderBuilder()
                    .WithDevice(device)
                    .WithLoopbackCapture()
                    .BuildAsync();

                // The mix format is the device's and cannot be negotiated in shared mode,
                // so it is read rather than requested — typically 32-bit float stereo at
                // 48 kHz, but an unusual sound card is exactly where guessing hurts.
                SampleRate = recorder.WaveFormat.SampleRate;
                DeviceName = recorder.DeviceFriendlyName ?? device.FriendlyName;

                recorder.DataAvailable += OnData;
                recorder.RecordingStopped += OnStopped;
                recorder.StartRecording();
                _recorder = recorder;

                Log.Write($"loopback open: {DeviceName} at {SampleRate} Hz, {recorder.WaveFormat.Channels} ch, " +
                          $"{recorder.WaveFormat.BitsPerSample}-bit {recorder.WaveFormat.Encoding}");
                return true;
            }
        }
        catch (Exception ex)
        {
            Log.Warn($"loopback capture unavailable: {ex.Message}");
            return false;
        }
    }

    public void Stop()
    {
        var recorder = _recorder;
        if (recorder is null) return;
        _recorder = null;

        try
        {
            recorder.DataAvailable -= OnData;
            recorder.RecordingStopped -= OnStopped;
            recorder.StopRecording();
            recorder.Dispose();
        }
        catch (Exception ex)
        {
            Log.Warn($"loopback stop failed: {ex.Message}");
        }
    }

    private void OnStopped(object? sender, StoppedEventArgs e)
    {
        if (e.Exception is not null) Log.Warn($"loopback stopped: {e.Exception.Message}");

        // Only interesting when we did not ask for it; Stop() detaches this first.
        if (_recorder is not null) Stopped?.Invoke();
    }

    private void OnData(ReadOnlySpan<byte> buffer, AudioClientBufferFlags flags, long position, long timestamp)
    {
        var format = _recorder?.WaveFormat;
        if (format is null || buffer.Length == 0) return;

        var channels = Math.Max(1, format.Channels);
        var bytesPerSample = format.BitsPerSample / 8;
        if (bytesPerSample == 0) return;

        var frames = buffer.Length / (bytesPerSample * channels);
        if (frames <= 0) return;

        if (_mono.Length < frames) _mono = new float[frames];

        FramesSeen += frames;
        if ((flags & AudioClientBufferFlags.DataDiscontinuity) != 0) Gaps++;
        if ((flags & AudioClientBufferFlags.Silent) != 0) SilentBuffers++;

        // WASAPI marks a buffer silent rather than bothering to zero it, so its contents
        // may be whatever was there last time. Trusting them would let a paused track go
        // on producing a beat.
        if ((flags & AudioClientBufferFlags.Silent) != 0)
        {
            Array.Clear(_mono, 0, frames);
            Frames?.Invoke(_mono, frames);
            return;
        }

        var isFloat = AudioSamples.IsFloat(format);

        for (var frame = 0; frame < frames; frame++)
        {
            float sum = 0;
            for (var channel = 0; channel < channels; channel++)
            {
                var at = (frame * channels + channel) * bytesPerSample;
                sum += AudioSamples.Read(buffer, at, bytesPerSample, isFloat);
            }

            // Downmixed rather than one channel taken: a track mixed hard to one side
            // would otherwise be half heard, or not heard at all.
            var value = sum / channels;
            _mono[frame] = value;

            if (Math.Abs(value) > _peak) _peak = Math.Abs(value);
            _energySum += value * (double)value;
            _frames++;
        }

        Frames?.Invoke(_mono, frames);
    }

    public void Dispose() => Stop();
}
```
