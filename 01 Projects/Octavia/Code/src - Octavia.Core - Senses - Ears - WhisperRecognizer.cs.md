---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Ears\WhisperRecognizer.cs
---

# src\Octavia.Core\Senses\Ears\WhisperRecognizer.cs

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

    private IAudioSource? _source;
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

    /// Mono 16 kHz frames as they arrive, for anything that wants to listen alongside the
    /// voice detector. Nothing is recorded here either.
    public event Action<float[], int>? Audio;

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
        ArmSilenceWatch();

        /* **No default source.** This used to open the machine's own microphone when nothing
           had been handed to it, which was the sensible fallback while the server had one.

           There is nothing to fall back to, and inventing one would be the exact failure
           Stage 14 kept finding: `UseSource` *starts* what it is given, so a default here
           would open a device because somebody let go of a button. Started with no source,
           the recogniser simply waits — and the log says so, because ears that are running
           and hearing nothing is precisely the state that looks like her ignoring you. */
        if (_source is null)
        {
            Log.Write("ears started with no source; waiting for a face to stream one");
            return;
        }

        // Detached first, so starting something already started subscribes once rather than
        // twice. Reachable since a face taking the floor calls this on ears the desk may
        // already have running, and the symptom would be every frame processed twice.
        _source.Data -= OnAudio;
        _source.Data += OnAudio;
        _source.Start();
    }

    public void Stop()
    {
        _wantListening = false;

        if (_source is not null)
        {
            _source.Data -= OnAudio;
            _source.Stop();
        }

        ResetUtterance();
    }

    /// Hands the ears a different microphone — a face's, rather than this machine's.
    ///
    /// The old source is detached but **not disposed**: the local microphone goes on
    /// running while a face holds the floor, because the room-music analyser is still
    /// listening to it. Speech moves rooms; her sense of what is playing around her does
    /// not. See ROADMAP.md stage 14 item 2.
    public void UseSource(IAudioSource source)
    {
        if (ReferenceEquals(source, _source)) return;

        /* Detached, and deliberately **not stopped**. The local microphone is shared: the
           room-music analyser frames it independently, so stopping it here would silence
           her sense of this room for exactly as long as somebody held the floor elsewhere
           — which is the trap this whole item is written around, wearing a different hat. */
        if (_source is not null) _source.Data -= OnAudio;

        _source = source;
        ResetUtterance();
        ArmSilenceWatch();

        Log.Write($"ears listening to {source.Name}");

        if (_wantListening)
        {
            _source.Data += OnAudio;
            _source.Start();
        }
    }

    private void ArmSilenceWatch()
    {
        _listeningSince = DateTime.UtcNow;
        _loudestHeard = 0f;
        _reportedDeaf = false;
    }

    /// End the utterance now and transcribe whatever has been gathered.
    ///
    /// For a source that knows where the sentence stopped. A released push-to-talk button
    /// is a far better end marker than 800 ms of quiet: it is exact, it costs no latency,
    /// and it cannot be fooled by someone pausing mid-thought. The voice detector still
    /// decides for the local microphone, which has nothing better to go on.
    ///
    /// **Returns whether a transcript is actually coming.** Transcription is asynchronous, so
    /// the caller has already moved on by the time the words exist — and a caller that needs
    /// to remember *who was speaking* has to know whether there is anything left to remember
    /// it for. Below `MinVoicedSeconds` there is not, and a latch set anyway would still be
    /// waiting when somebody else spoke next.
    public bool Flush()
    {
        float[]? samples;

        lock (_utterance)
        {
            var voicedSeconds = _voicedFrames * SileroVad.FrameSamples / (float)SileroVad.SampleRate;
            samples = voicedSeconds >= MinVoicedSeconds ? _utterance.ToArray() : null;

            _utterance.Clear();
            _preroll.Clear();
            _inSpeech = false;
            _quietFrames = 0;
            _voicedFrames = 0;
            _frameFill = 0;
            _vad.Reset();
        }

        if (samples is null)
        {
            // "Press" would be a lie half the time now: a room switching its listening off
            // flushes through here too, and there was never a button.
            Log.Write("the utterance ended with nothing voiced in it");
            return false;
        }

        Hypothesised?.Invoke("…");
        _ = TranscribeAsync(samples, asked: true);
        return true;
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

    private void OnAudio(byte[] buffer, int count)
    {
        if (_muted || !_wantListening || _disposed) return;

        WatchForSilence(buffer, count);

        for (var i = 0; i < count - 1; i += 2)
        {
            _frame[_frameFill++] = (short)(buffer[i] | (buffer[i + 1] << 8)) / 32768f;
            if (_frameFill < SileroVad.FrameSamples) continue;

            _frameFill = 0;

            /* The same frames, offered to anyone else who wants them — the music analyser,
               so she can hear a room and not only this machine. Raised outside the lock
               below on purpose: a subscriber doing real work must not hold up the voice
               detector, and the analyser copies what it needs. */
            Audio?.Invoke(_frame, SileroVad.FrameSamples);

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

    private void WatchForSilence(byte[] buffer, int count)
    {
        if (_reportedDeaf) return;

        /* Only for a source that is supposed to be delivering continuously. A local device
           that is open and handing over digital zeroes is broken, and saying so is one of
           the more useful things she does. A push-to-talk face delivering nothing is
           somebody **not holding a button** — the normal state — and warning about it would
           name Remote Desktop audio settings at a person holding a phone. */
        if (_source?.ExpectsContinuousAudio != true) return;

        for (var i = 0; i < count - 1; i += 2)
        {
            var sample = Math.Abs((short)(buffer[i] | (buffer[i + 1] << 8)) / 32768f);
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

    /// <param name="asked">
    /// Whether somebody deliberately ended this utterance, as opposed to the voice detector
    /// deciding they had stopped talking.
    /// </param>
    ///
    /// <remarks>
    /// **`_muted` and `_wantListening` guard incoming audio, not words already in flight, and
    /// conflating the two threw away every sentence ever spoken into a phone.**
    ///
    /// Releasing the button calls `ReleaseFloor`, which flushes — starting this — and then,
    /// when nobody at the desk is listening, calls `Stop()` on the very next line. `Stop`
    /// clears `_wantListening`. Transcription finishes a second or two later, finds the flag
    /// down, and returns without a word to anyone. The floor was taken, the audio arrived,
    /// Whisper ran, and the log showed a press followed by nothing at all.
    ///
    /// A flushed utterance is somebody's deliberate act and is delivered whatever the flags
    /// have done since. The detector's own segments still respect them: those are the room
    /// being overheard, which is exactly what the flags exist to stop.
    /// </remarks>
    private async Task TranscribeAsync(float[] samples, bool asked = false)
    {
        try
        {
            var result = await _whisper.TranscribeAsync(samples);

            if (result.Text.Length == 0 || !result.Text.Any(char.IsLetterOrDigit))
            {
                // Never silently, now. "I pressed the button and nothing happened" was
                // indistinguishable from four other failures, all of them mute.
                if (asked) Log.Write("the utterance was flushed but Whisper heard no words in it");
                return;
            }

            if (!asked && (_muted || !_wantListening)) return;

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
