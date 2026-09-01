---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\NeuralVoice.cs
---

# src\Octavia.Core\Voice\NeuralVoice.cs

```csharp
using System.Diagnostics;
using NAudio.Wave;
using Octavia.Audio;
using Octavia.Core;

namespace Octavia.Voice;

/// A neural voice, out of process.
///
/// Same reasoning as the local brain: a second ONNX runtime inside this process would sit
/// beside Whisper's CUDA-linked one, and native dependency collisions are not worth the
/// milliseconds saved. Piper is a long-lived child process — started once, sentences
/// written to its input, raw audio read from its output — so the 60 MB model loads once
/// rather than once per sentence.
internal sealed class NeuralVoice : IVoice
{
    /// Playback drained and nothing new arriving for this long ends the utterance. Piper
    /// gives no end-of-sentence marker, so quiet is the only signal there is.
    private static readonly TimeSpan Settled = TimeSpan.FromMilliseconds(320);

    private readonly OctaviaConfig _config;
    private readonly object _gate = new();

    private Process? _piper;
    private WaveOut? _output;
    private BufferedWaveProvider? _buffer;
    private VisemeReader? _reader;
    private Thread? _pump;
    private System.Threading.Timer? _watchdog;

    private string _voice = "";
    private int _sampleRate = 22050;
    private bool _speaking;
    private bool _discarding;
    private volatile bool _awaitingAudio;
    private DateTime _lastAudio = DateTime.MinValue;
    private string? _lastShape;
    private double _lastOpenness = -1;
    private bool _disposed;

    /// Read on the sound card's thread and written from the turn, so `volatile`. See
    /// `IVoice.Aloud` and `MouthTap`: the audio is still produced, still teed and still
    /// read for visemes — only the buffer handed to the speakers is emptied.
    private volatile bool _aloud = true;

    public bool Aloud
    {
        get => _aloud;
        set => _aloud = value;
    }

    public event Action<double, string?>? Viseme;
    public event Action? Started;
    public event Action? Finished;
    public event Action<ReadOnlyMemory<byte>>? Audio;

    /// Read from the voice model's own config, so it follows the voice rather than being
    /// a constant. A face must re-read it on every `hello`.
    public AudioFormat? AudioFormat => new(_sampleRate, 16, 1);
    public event Action<string>? Trouble;

    public bool IsSpeaking => _speaking;
    public string EngineName => $"Piper ({PiperStore.Pretty(_voice)})";
    public string CurrentVoice => _voice;

    public NeuralVoice(OctaviaConfig config) => _config = config;

    /// Downloads what is missing and starts the engine. Separate from the constructor
    /// because it can take a minute on a first run and must not block the message loop.
    public async Task StartAsync(string voice, CancellationToken cancel = default)
    {
        await PiperStore.EnsureAsync(voice, message => Trouble?.Invoke(message), cancel);
        Launch(voice);
    }

    public IReadOnlyList<string> InstalledVoices()
    {
        // What is on disk first, then the rest of the shortlist — so the menu offers
        // more than what has already been fetched, without hiding what is ready now.
        var downloaded = PiperStore.Downloaded();
        return downloaded.Concat(PiperStore.Catalogue.Except(downloaded)).ToList();
    }

    /// Switching voice restarts the engine: a Piper process is bound to one model.
    public bool SelectVoice(string? name)
    {
        if (string.IsNullOrWhiteSpace(name) || name == _voice) return false;
        if (!PiperStore.HasVoice(name))
        {
            // Fetch it in the background rather than blocking whoever chose it.
            Task.Run(async () =>
            {
                try { await StartAsync(name); }
                catch (Exception ex)
                {
                    Log.Error($"could not switch to neural voice '{name}'", ex);
                    Trouble?.Invoke($"Could not fetch the voice '{PiperStore.Pretty(name)}'.");
                }
            }).Forget("neural voice switch");
            return false;
        }

        Launch(name);
        return true;
    }

    private void Launch(string voice)
    {
        lock (_gate)
        {
            StopEngine();

            _sampleRate = ReadSampleRate(PiperStore.ConfigPath(voice));
            _voice = voice;

            var start = new ProcessStartInfo(PiperStore.EnginePath)
            {
                WorkingDirectory = Path.GetDirectoryName(PiperStore.EnginePath)!,
                RedirectStandardInput = true,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,
                CreateNoWindow = true
            };
            start.ArgumentList.Add("--model");
            start.ArgumentList.Add(PiperStore.ModelPath(voice));
            start.ArgumentList.Add("--config");
            start.ArgumentList.Add(PiperStore.ConfigPath(voice));
            start.ArgumentList.Add("--output_raw");

            // Her speaking rate, in the only units Piper has: how long each phoneme is
            // held. Slower than 1 is a longer phoneme, so the scale runs backwards.
            var rate = Math.Clamp(_config.VoiceRate, -10, 10);
            start.ArgumentList.Add("--length_scale");
            start.ArgumentList.Add((1.0 - rate * 0.03).ToString("0.00", System.Globalization.CultureInfo.InvariantCulture));

            _piper = Process.Start(start) ?? throw new InvalidOperationException("the speech engine would not start");

            // Piper narrates to stderr; draining it stops the pipe filling and the
            // process wedging halfway through a sentence.
            _piper.ErrorDataReceived += (_, e) => { if (e.Data is { Length: > 0 }) Log.Debug($"piper: {e.Data}"); };
            _piper.BeginErrorReadLine();

            _reader = new VisemeReader(_sampleRate);
            // NAudio 3 takes the buffer length as a constructor argument; it is not a
            // settable property any more.
            _buffer = new BufferedWaveProvider(new WaveFormat(_sampleRate, 16, 1), TimeSpan.FromSeconds(30))
            {
                DiscardOnBufferOverflow = true
            };

            _output = new WaveOut { BufferMilliseconds = 80, NumberOfBuffers = 3 };
            _output.Init(new MouthTap(_buffer, OnAudioPlayed, () => _aloud));
            _output.Play();

            _pump = new Thread(Pump) { IsBackground = true, Name = "octavia-piper" };
            _pump.Start();

            _watchdog = new System.Threading.Timer(_ => CheckSettled(), null, 200, 120);

            Log.Write($"neural voice: {PiperStore.Pretty(voice)} at {_sampleRate} Hz");
        }
    }

    /// Reads raw PCM out of the engine as fast as it is produced. Synthesis runs ahead of
    /// playback once warm, which is exactly what makes her sound continuous.
    private void Pump()
    {
        var piper = _piper;
        var buffer = new byte[8192];

        try
        {
            var stream = piper?.StandardOutput.BaseStream;
            while (stream is not null && !_disposed)
            {
                var read = stream.Read(buffer, 0, buffer.Length);
                if (read <= 0) break;

                _lastAudio = DateTime.UtcNow;

                // After a hush the engine keeps finishing its sentence; that audio is
                // read and thrown away rather than restarting the process for it.
                if (_discarding) continue;
                _awaitingAudio = false;
                _buffer?.AddSamples(buffer, 0, read);
            }
        }
        catch (Exception ex) when (!_disposed)
        {
            Log.Error("the neural voice stopped reading", ex);
            Trouble?.Invoke("Her voice engine stopped. Falling back needs a restart.");
        }
    }

    /// Called with the audio at the moment it is handed to the sound card, so the mouth
    /// is in step with what is heard rather than with what has been generated.
    private void OnAudioPlayed(ReadOnlySpan<byte> pcm)
    {
        /* Tee to any face that asked for her voice, before the visemes are read, so the
           two leave from the same buffer at the same instant.

           The span **must be copied**: it cannot be captured by an async send, and it is a
           view over a buffer that is about to be reused. Rented rather than allocated,
           because at forty frames a second the garbage would be constant, and returned
           the moment the handlers are done — which is why the event contract says a
           handler may not hold it. */
        if (Audio is { } audio && pcm.Length > 0)
        {
            var copy = System.Buffers.ArrayPool<byte>.Shared.Rent(pcm.Length);

            try
            {
                pcm.CopyTo(copy);
                audio(new ReadOnlyMemory<byte>(copy, 0, pcm.Length));
            }
            finally
            {
                System.Buffers.ArrayPool<byte>.Shared.Return(copy);
            }
        }

        var reader = _reader;
        if (reader is null) return;

        var samples = System.Runtime.InteropServices.MemoryMarshal.Cast<byte, short>(pcm);
        for (var offset = 0; offset + VisemeReader.FrameSamples <= samples.Length; offset += VisemeReader.FrameSamples)
        {
            var mouth = reader.Read(samples.Slice(offset, VisemeReader.FrameSamples));

            // The sound card is fed continuously and a buffer with nothing in it comes
            // back as silence, so without this she would send a viseme twelve times a
            // second forever — every one of them saying "mouth shut".
            if (mouth.Shape == _lastShape && Math.Abs(mouth.Openness - _lastOpenness) < 0.02) continue;

            _lastShape = mouth.Shape;
            _lastOpenness = mouth.Openness;
            Viseme?.Invoke(mouth.Openness, mouth.Shape);
        }
    }

    public void Say(string sentence)
    {
        if (_disposed || string.IsNullOrWhiteSpace(sentence)) return;

        lock (_gate)
        {
            if (_piper is null || _piper.HasExited)
            {
                Trouble?.Invoke("Her voice engine is not running.");
                return;
            }

            _discarding = false;
            _awaitingAudio = true;
            _lastAudio = DateTime.UtcNow;

            if (!_speaking)
            {
                _speaking = true;
                Started?.Invoke();
            }

            try
            {
                // One line in, one utterance out. Newlines inside would be two.
                _piper.StandardInput.WriteLine(sentence.Replace('\n', ' ').Replace('\r', ' '));
                _piper.StandardInput.Flush();
            }
            catch (Exception ex)
            {
                Log.Error("could not hand a sentence to the voice engine", ex);
                Trouble?.Invoke("Her voice engine stopped accepting text.");
            }
        }
    }

    public void Hush()
    {
        if (_disposed) return;

        lock (_gate)
        {
            _discarding = true;
            _awaitingAudio = false;
            _buffer?.ClearBuffer();
            _speaking = false;
        }

        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    /// Piper marks nothing, so the end of an utterance is quiet: the playback buffer
    /// empty and nothing new arriving for a moment.
    private void CheckSettled()
    {
        if (!_speaking || _disposed) return;

        // Synthesis takes a moment to produce its first sample, and during that moment
        // the buffer is empty and nothing has arrived — which looks exactly like having
        // finished. She would go idle and then talk. Nothing is over until it has begun.
        if (_awaitingAudio) return;

        var quiet = DateTime.UtcNow - _lastAudio > Settled;
        var drained = (_buffer?.BufferedBytes ?? 0) == 0;
        if (!quiet || !drained) return;

        lock (_gate)
        {
            if (!_speaking) return;
            _speaking = false;
            _discarding = false;
        }

        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    private static int ReadSampleRate(string configPath)
    {
        try
        {
            using var document = System.Text.Json.JsonDocument.Parse(File.ReadAllText(configPath));
            if (document.RootElement.TryGetProperty("audio", out var audio) &&
                audio.TryGetProperty("sample_rate", out var rate))
                return rate.GetInt32();
        }
        catch (Exception ex)
        {
            Log.Warn($"could not read the voice's sample rate, assuming 22050: {ex.Message}");
        }

        return 22050;
    }

    private void StopEngine()
    {
        _watchdog?.Dispose();
        _watchdog = null;

        try { _output?.Stop(); } catch (Exception ex) { Log.Debug($"voice stop: {ex.Message}"); }
        _output?.Dispose();
        _output = null;
        _buffer = null;

        if (_piper is not null)
        {
            try
            {
                if (!_piper.HasExited)
                {
                    _piper.StandardInput.Close();
                    if (!_piper.WaitForExit(1200)) _piper.Kill(entireProcessTree: true);
                }
            }
            catch (Exception ex) { Log.Debug($"voice engine teardown: {ex.Message}"); }

            _piper.Dispose();
            _piper = null;
        }

        _pump = null;
        _speaking = false;
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        lock (_gate) StopEngine();
    }
}

/// Passes audio through to the sound card and shows it to the mouth on the way.
///
/// The tap is here rather than where the audio is produced because synthesis runs ahead
/// of playback: reading visemes off the generator would have her mouth finish a sentence
/// a second before it was heard.
///
/// `aloud` is asked *after* the tap and never before it. That order is the whole design:
/// when she is answering another room the visemes and the streamed PCM must be exactly
/// what they would have been, and only the speakers go quiet. Zeroing rather than
/// returning 0 keeps playback running at real time, so her mouth, her state and the
/// audio a phone is receiving all stay on the same clock.
internal sealed class MouthTap(
    IWaveProvider source, Action<ReadOnlySpan<byte>> tap, Func<bool> aloud) : IWaveProvider
{
    public WaveFormat WaveFormat => source.WaveFormat;

    public int Read(Span<byte> buffer)
    {
        var read = source.Read(buffer);
        if (read <= 0) return read;

        tap(buffer[..read]);
        if (!aloud()) buffer[..read].Clear();
        return read;
    }
}
```
