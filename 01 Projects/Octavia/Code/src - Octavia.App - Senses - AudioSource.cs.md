---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\AudioSource.cs
---

# src\Octavia.App\Senses\AudioSource.cs

```csharp
using NAudio.Wave;
using Octavia.Core;

namespace Octavia.Senses;

/// Where 16 kHz, 16-bit, mono, little-endian PCM comes from.
///
/// This exists because `WhisperRecognizer` did not *consume* microphone audio — it **was**
/// the microphone. It built its own `WaveIn`, chose the device, and raised `DataAvailable`
/// into its own frame loop, so there was nothing to hand a different source to. Everything
/// above that line was already source-agnostic: bytes become 512-sample frames and the
/// voice detector never cared where they came from.
///
/// **The format is fixed by contract, not negotiated.** 16 kHz is what Silero and Whisper
/// want, and a handset has cycles to spare for the resample — the host should not grow a
/// resampler to save a phone some work.
internal interface IAudioSource : IDisposable
{
    /// A buffer and how much of it is real, matching `WaveInEventArgs` so the existing
    /// frame loop needed no changes at all.
    event Action<byte[], int>? Data;

    /// What this is, for the log and for the face: a device name, or a face.
    string Name { get; }

    /// Whether silence from this source is worth complaining about.
    ///
    /// A local device that is open and delivering digital zeroes is broken, and saying so
    /// is one of the more valuable things she does. A push-to-talk face delivering nothing
    /// is **somebody not holding a button**, which is the normal state — warning about it
    /// would name RDP settings at a person holding a phone.
    bool ExpectsContinuousAudio { get; }

    void Start();
    void Stop();
}

/// The microphone attached to this machine. Lifted out of `WhisperRecognizer` unchanged.
internal sealed class LocalMicSource(string? device) : IAudioSource
{
    private WaveIn? _capture;
    private bool _disposed;

    public event Action<byte[], int>? Data;

    public string Name { get; private set; } = "the Windows default";
    public bool ExpectsContinuousAudio => true;

    public void Start()
    {
        if (_capture is not null || _disposed) return;

        _capture = new WaveIn
        {
            WaveFormat = new WaveFormat(SileroVad.SampleRate, 16, 1),
            BufferMilliseconds = 32
        };

        var index = AudioDevices.WaveInIndex(device);
        if (index >= 0)
        {
            _capture.DeviceNumber = index;
            Name = WaveIn.GetCapabilities(index).ProductName;
            Log.Write($"listening on '{Name}'");
        }

        _capture.DataAvailable += Forward;
        _capture.StartRecording();
    }

    private void Forward(object? sender, WaveInEventArgs e) => Data?.Invoke(e.Buffer, e.BytesRecorded);

    public void Stop()
    {
        var capture = _capture;
        _capture = null;
        if (capture is null) return;

        try
        {
            capture.DataAvailable -= Forward;
            capture.StopRecording();
            capture.Dispose();
        }
        catch (Exception ex)
        {
            Log.Write($"stopping the microphone failed: {ex.Message}");
        }
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        Stop();
    }
}

/// A microphone on another device, pushed in over the socket a frame at a time.
///
/// It owns no hardware and opens nothing: `Push` is called by whoever is reading binary
/// frames from a face. `Start` and `Stop` only decide whether those frames are passed on,
/// which is what makes handing the floor between sources a matter of who is listening
/// rather than what is open.
internal sealed class FaceAudioSource : IAudioSource
{
    private volatile bool _running;

    public event Action<byte[], int>? Data;

    public string Name { get; }
    public bool ExpectsContinuousAudio => false;

    public FaceAudioSource(string name) => Name = name;

    public void Push(byte[] pcm, int count)
    {
        if (_running) Data?.Invoke(pcm, count);
    }

    public void Start() => _running = true;
    public void Stop() => _running = false;
    public void Dispose() => _running = false;
}

/// Turns a byte stream into the 512-sample float frames everything downstream expects.
///
/// Extracted so more than one thing can frame the *same* source independently. That is not
/// tidiness: the room-music analyser has to keep hearing **this** room while a face holds
/// the floor for speech, so it frames the local microphone on its own rather than taking
/// whatever the recogniser happens to be consuming. See ROADMAP.md stage 14 item 2.
internal sealed class PcmFramer
{
    private readonly float[] _frame = new float[SileroVad.FrameSamples];
    private int _fill;

    /// A full frame, and how many samples of it are real (always the whole frame).
    public event Action<float[], int>? Frame;

    public void Push(byte[] buffer, int count)
    {
        for (var i = 0; i < count - 1; i += 2)
        {
            _frame[_fill++] = (short)(buffer[i] | (buffer[i + 1] << 8)) / 32768f;
            if (_fill < SileroVad.FrameSamples) continue;

            _fill = 0;
            Frame?.Invoke(_frame, SileroVad.FrameSamples);
        }
    }
}
```
