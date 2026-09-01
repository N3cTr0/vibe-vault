---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\MicLevelMeter.cs
---

# src\Octavia.Core\Senses\MicLevelMeter.cs

```csharp
using NAudio.Wave;
using Octavia.Core;

namespace Octavia.Senses;

/// Reads the default capture device purely for amplitude. This is what makes the face
/// react while you are still mid-sentence, and it is the seam that WASAPI loopback
/// (reacting to music) will eventually plug into.
internal sealed class MicLevelMeter : IDisposable
{
    private WaveIn? _capture;
    private float _level;

    /// Which capture device to meter. Empty follows the Windows default. Set before
    /// Start; it is read each time the capture opens.
    public string? Device { get; set; }

    public event Action<float>? LevelChanged;

    public float Level => _level;
    public bool IsRunning => _capture is not null;

    public void Start()
    {
        if (_capture is not null) return;

        try
        {
            _capture = new WaveIn
            {
                WaveFormat = new WaveFormat(16000, 16, 1),
                BufferMilliseconds = 50
            };

            // The same device the ears use, or the meter would animate to one
            // microphone while she transcribed another.
            var index = AudioDevices.WaveInIndex(Device);
            if (index >= 0) _capture.DeviceNumber = index;

            _capture.DataAvailable += OnData;
            _capture.StartRecording();
        }
        catch (Exception ex)
        {
            Log.Write($"level meter unavailable: {ex.Message}");
            Dispose();
        }
    }

    public void Stop()
    {
        if (_capture is null) return;

        try
        {
            _capture.DataAvailable -= OnData;
            _capture.StopRecording();
            _capture.Dispose();
        }
        catch (Exception ex)
        {
            Log.Write($"level meter stop failed: {ex.Message}");
        }

        _capture = null;
        _level = 0f;
        LevelChanged?.Invoke(0f);
    }

    private void OnData(object? sender, WaveInEventArgs e)
    {
        if (e.BytesRecorded < 2) return;

        double sum = 0;
        var samples = e.BytesRecorded / 2;
        for (var i = 0; i < e.BytesRecorded - 1; i += 2)
        {
            var sample = (short)(e.Buffer[i] | (e.Buffer[i + 1] << 8)) / 32768.0;
            sum += sample * sample;
        }

        var rms = Math.Sqrt(sum / samples);
        var scaled = (float)Math.Clamp(rms * 5.5, 0, 1);

        // Rise fast so consonants register, fall slow so the face doesn't strobe.
        _level = scaled > _level ? scaled : _level + (scaled - _level) * 0.25f;
        LevelChanged?.Invoke(_level);
    }

    public void Dispose() => Stop();
}
```
