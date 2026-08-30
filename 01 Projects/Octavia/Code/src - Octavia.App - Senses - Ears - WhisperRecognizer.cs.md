---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Ears\WhisperRecognizer.cs
---

# src\Octavia.App\Senses\Ears\WhisperRecognizer.cs

```csharp
using NAudio.Wave;
using Octavia.Core;

namespace Octavia.Senses;

/// Local ears: mic → Silero VAD → Whisper. Nothing leaves the machine.
/// The VAD decides where an utterance starts and ends; Whisper only ever sees
/// audio the VAD vouched for.
internal sealed class WhisperRecognizer : ISpeechRecognizer
{
    private const float StartThreshold = 0.5f;
    private const float EndThreshold = 0.35f;
    private const int PrerollFrames = 10;        // 320ms kept from before speech began
    private const int HangoverFrames = 25;       // 800ms of quiet ends the utterance
    private const float MinVoicedSeconds = 0.3f;
    private const float MaxUtteranceSeconds = 30f;

    private readonly SileroVad _vad;
    private readonly WhisperTranscriber _whisper;
    private readonly string _modelName;
    private readonly string? _device;

    private WaveIn? _capture;
    private readonly float[] _frame = new float[SileroVad.FrameSamples];
    private int _frameFill;
    private readonly Queue<float[]> _preroll = new();
    private readonly List<float> _utterance = [];
    private bool _inSpeech;
    private int _quietFrames;
    private int _voicedFrames;

    private volatile bool _wantListening;
    private volatile bool _muted;
    private bool _disposed;

    public event Action<string, float>? Recognized;
    public event Action<string>? Hypothesised;
    public event Action<string>? Trouble;

    // A device can open successfully and still deliver digital silence — a muted
    // input, or RDP with microphone redirection switched off. Without this the
    // failure is invisible: she looks like she is listening and simply never answers.
    private const float SilenceFloor = 0.0005f;
    private static readonly TimeSpan DeafAfter = TimeSpan.FromSeconds(10);
    private DateTime _listeningSince = DateTime.MaxValue;
    private float _loudestHeard;
    private bool _reportedDeaf;

    public string EngineName => $"Whisper {_modelName} (local)";
    public bool IsListening => _wantListening && !_muted;

    public WhisperRecognizer(string modelPath, string modelName, string language, string? compute = null, int threads = 0, string? device = null)
    {
        _modelName = modelName;
        _device = device;
        _vad = new SileroVad(WhisperModelStore.SileroPath);
        _whisper = new WhisperTranscriber(modelPath, language, compute, threads);
    }

    public void Start()
    {
        _wantListening = true;
        _listeningSince = DateTime.UtcNow;
        _loudestHeard = 0f;
        _reportedDeaf = false;
        if (_capture is not null) return;

        _capture = new WaveIn
        {
            WaveFormat = new WaveFormat(SileroVad.SampleRate, 16, 1),
            BufferMilliseconds = 32
        };

        var index = AudioDevices.WaveInIndex(_device);
        if (index >= 0)
        {
            _capture.DeviceNumber = index;
            Log.Write($"listening on '{WaveIn.GetCapabilities(index).ProductName}'");
        }

        _capture.DataAvailable += OnAudio;
        _capture.RecordingStopped += (_, e) =>
        {
            if (e.Exception is not null) Log.Write($"mic stopped: {e.Exception.Message}");
        };
        _capture.StartRecording();
    }

    public void Stop()
    {
        _wantListening = false;
        var capture = _capture;
        _capture = null;

        if (capture is not null)
        {
            capture.DataAvailable -= OnAudio;
            try { capture.StopRecording(); } catch (Exception ex) { Log.Write($"mic stop: {ex.Message}"); }
            capture.Dispose();
        }

        ResetUtterance();
    }

    public void Mute()
    {
        _muted = true;
        ResetUtterance();
    }

    public void Unmute() => _muted = false;

    private void ResetUtterance()
    {
        lock (_utterance)
        {
            _utterance.Clear();
            _preroll.Clear();
            _inSpeech = false;
            _quietFrames = 0;
            _voicedFrames = 0;
            _frameFill = 0;
            _vad.Reset();
        }
    }

    private void OnAudio(object? sender, WaveInEventArgs e)
    {
        if (_muted || !_wantListening || _disposed) return;

        WatchForSilence(e);

        for (var i = 0; i < e.BytesRecorded - 1; i += 2)
        {
            _frame[_frameFill++] = (short)(e.Buffer[i] | (e.Buffer[i + 1] << 8)) / 32768f;
            if (_frameFill < SileroVad.FrameSamples) continue;

            _frameFill = 0;
            try
            {
                ProcessFrame();
            }
            catch (Exception ex)
            {
                Log.Write($"vad frame failed: {ex.Message}");
            }
        }
    }

    private void WatchForSilence(WaveInEventArgs e)
    {
        if (_reportedDeaf) return;

        for (var i = 0; i < e.BytesRecorded - 1; i += 2)
        {
            var sample = Math.Abs((short)(e.Buffer[i] | (e.Buffer[i + 1] << 8)) / 32768f);
            if (sample > _loudestHeard) _loudestHeard = sample;
        }

        if (_loudestHeard > SilenceFloor)
        {
            _listeningSince = DateTime.MaxValue;
            return;
        }

        if (DateTime.UtcNow - _listeningSince < DeafAfter) return;

        _reportedDeaf = true;
        Log.Write($"microphone open but silent for {DeafAfter.TotalSeconds:0}s");
        Trouble?.Invoke(
            "The microphone is open but completely silent. Over RDP, set Local Resources " +
            "> Remote audio > Settings to 'Record from this computer', then reconnect.");
    }

    private void ProcessFrame()
    {
        float probability;
        lock (_utterance)
        {
            probability = _vad.Probability(_frame);

            if (!_inSpeech)
            {
                if (probability >= StartThreshold)
                {
                    _inSpeech = true;
                    _quietFrames = 0;
                    _voicedFrames = 1;
                    foreach (var kept in _preroll) _utterance.AddRange(kept);
                    _preroll.Clear();
                    _utterance.AddRange(_frame);
                }
                else
                {
                    _preroll.Enqueue((float[])_frame.Clone());
                    while (_preroll.Count > PrerollFrames) _preroll.Dequeue();
                }
                return;
            }

            _utterance.AddRange(_frame);
            if (probability >= EndThreshold)
            {
                _voicedFrames++;
                _quietFrames = 0;
            }
            else
            {
                _quietFrames++;
            }

            var seconds = _utterance.Count / (float)SileroVad.SampleRate;
            if (_quietFrames < HangoverFrames && seconds < MaxUtteranceSeconds) return;

            var voicedSeconds = _voicedFrames * SileroVad.FrameSamples / (float)SileroVad.SampleRate;
            var samples = voicedSeconds >= MinVoicedSeconds ? _utterance.ToArray() : null;

            _utterance.Clear();
            _inSpeech = false;
            _quietFrames = 0;
            _voicedFrames = 0;

            if (samples is null) return;

            Hypothesised?.Invoke("…");
            _ = TranscribeAsync(samples);
        }
    }

    private async Task TranscribeAsync(float[] samples)
    {
        try
        {
            var result = await _whisper.TranscribeAsync(samples);
            if (_muted || !_wantListening || result.Text.Length == 0) return;
            if (!result.Text.Any(char.IsLetterOrDigit)) return;

            Recognized?.Invoke(result.Text, result.Confidence);
        }
        catch (Exception ex)
        {
            Log.Write($"transcription failed: {ex.Message}");
        }
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        Stop();
        _whisper.Dispose();
        _vad.Dispose();
    }
}
```
