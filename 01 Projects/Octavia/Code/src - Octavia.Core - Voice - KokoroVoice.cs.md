---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\KokoroVoice.cs
---

# src\Octavia.Core\Voice\KokoroVoice.cs

```csharp
using System.Diagnostics;
using System.Runtime.InteropServices;
using NAudio.Wave;
using Octavia.Audio;
using Octavia.Core;

namespace Octavia.Voice;

/// Her voice.
///
/// Not *a* voice — **the** voice. Twenty-two candidates read the same paragraph in Stage 16
/// and the owner picked `af_heart` by ear, so the engine, the model and the speaker are all
/// constants here rather than settings. There is nothing to choose, and `IVoice` no longer
/// offers a choice: see the note on `KokoroStore`.
///
/// It runs **out of process**, for the reason Piper did before it and the local brain still
/// does: sherpa-onnx carries its own native `onnxruntime.dll` and this assembly carries
/// Microsoft's for Silero and the wake word. Two of those in one output folder is a native
/// collision, and no number of saved milliseconds is worth an afternoon of diagnosing it.
/// `Octavia.Kokoro` is that process; the contract between them is written at the top of its
/// `Program.cs`.
internal sealed class KokoroVoice : IVoice
{
    /// Playback drained and nothing new arriving for this long ends the utterance. The
    /// engine marks nothing, so quiet is the only signal there is.
    private static readonly TimeSpan Settled = TimeSpan.FromMilliseconds(320);

    /// A control word rather than something to say. Speech text has its control characters
    /// stripped before it is written, so this cannot be collided with by anything she means.
    private const char Control = (char)1;

    private readonly OctaviaConfig _config;
    private readonly object _gate = new();

    /// Samples left over from the last chunk, waiting to make a whole viseme frame.
    private readonly List<short> _carry = new(VisemeReader.FrameSamples * 2);

    private Process? _engine;
    private Pacer? _pacer;
    private BufferedWaveProvider? _buffer;
    private VisemeReader? _reader;
    private Thread? _pump;
    private System.Threading.Timer? _watchdog;

    private int _sampleRate = KokoroStore.SampleRate;
    private bool _speaking;
    private volatile bool _awaitingAudio;
    private DateTime _lastAudio = DateTime.MinValue;
    private string? _lastShape;
    private double _lastOpenness = -1;
    private bool _disposed;

    /// Read on the clock's thread and written from the turn, so `volatile`. Permanently
    /// false in practice — the server has no speakers for it to silence — and kept because
    /// `IVoice` still carries it; see `OctaviaSession`, where both assignments explain
    /// themselves.
    private volatile bool _aloud;

    public bool Aloud
    {
        get => _aloud;
        set => _aloud = value;
    }

    public event Action<double, string?>? Viseme;
    public event Action? Started;
    public event Action? Finished;
    public event Action<ReadOnlyMemory<byte>>? Audio;
    public event Action<string>? Trouble;

    /// Read from the engine's own greeting rather than assumed, so it follows the model.
    /// A face must re-read it on every `hello`.
    public AudioFormat? AudioFormat => new(_sampleRate, 16, 1);

    public bool IsSpeaking => _speaking;
    public string EngineName => $"Kokoro ({KokoroStore.Voice})";

    public KokoroVoice(OctaviaConfig config) => _config = config;

    /// Downloads what is missing and starts the engine. Separate from the constructor
    /// because a first run fetches 350 MB and must not block the message loop.
    public async Task StartAsync(CancellationToken cancel = default)
    {
        await KokoroStore.EnsureAsync(message => Trouble?.Invoke(message), cancel);
        Launch();
    }

    private void Launch()
    {
        lock (_gate)
        {
            StopEngine();

            var exe = KokoroStore.EnginePath()
                ?? throw new FileNotFoundException(
                    "octavia-kokoro.exe is not beside her or in a working tree. See README.md — it is a third publish.");

            var start = new ProcessStartInfo(exe)
            {
                WorkingDirectory = Path.GetDirectoryName(exe)!,
                RedirectStandardInput = true,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,
                CreateNoWindow = true,

                // Her text is prose, and prose has accents in it. The default encoding for a
                // redirected pipe is the console's code page, which would mangle them
                // silently — the words would still be said, just not the ones written.
                StandardInputEncoding = new System.Text.UTF8Encoding(false)
            };

            start.ArgumentList.Add("--model-dir");
            start.ArgumentList.Add(KokoroStore.ModelDir);
            start.ArgumentList.Add("--sid");
            start.ArgumentList.Add(KokoroStore.Speaker.ToString());
            start.ArgumentList.Add("--speed");
            start.ArgumentList.Add(Speed().ToString("0.00", System.Globalization.CultureInfo.InvariantCulture));

            _engine = Process.Start(start) ?? throw new InvalidOperationException("her voice engine would not start");

            /* It narrates to stderr, and draining that is not optional: a full pipe wedges
               the writer, which here means she stops mid-sentence for no visible reason.
               The greeting also carries the model's real sample rate, which is the one
               number in this file worth taking from the engine rather than assuming. */
            _engine.ErrorDataReceived += (_, e) =>
            {
                if (e.Data is not { Length: > 0 }) return;
                Log.Debug($"kokoro: {e.Data}");

                var at = e.Data.IndexOf(" Hz", StringComparison.Ordinal);
                if (at <= 0 || !e.Data.StartsWith("kokoro ready:", StringComparison.Ordinal)) return;

                var digits = e.Data.LastIndexOf(' ', at - 1) + 1;
                if (int.TryParse(e.Data.AsSpan(digits, at - digits), out var rate) && rate != _sampleRate)
                    Log.Warn($"her voice is {rate} Hz, not the {_sampleRate} Hz assumed; restart to pick it up");
            };
            _engine.BeginErrorReadLine();

            _reader = new VisemeReader(_sampleRate);

            // NAudio 3 takes the buffer length as a constructor argument; it is not a
            // settable property any more.
            _buffer = new BufferedWaveProvider(new WaveFormat(_sampleRate, 16, 1), TimeSpan.FromSeconds(30))
            {
                DiscardOnBufferOverflow = true
            };

            _pacer = new Pacer(_buffer, OnAudioPlayed);

            _pump = new Thread(Pump) { IsBackground = true, Name = "octavia-kokoro" };
            _pump.Start();

            _watchdog = new System.Threading.Timer(_ => CheckSettled(), null, 200, 120);

            Log.Write($"her voice: {EngineName} at {_sampleRate} Hz");
        }
    }

    /// `VoiceRate` runs -10 to 10 and means *faster* as it rises. Kokoro takes a multiplier,
    /// which runs the same way — unlike Piper's `--length_scale`, which was how long a
    /// phoneme is *held* and so ran backwards. An existing `config.json` keeps working and
    /// means what it looks like.
    private double Speed() => 1.0 + Math.Clamp(_config.VoiceRate, -10, 10) * 0.03;

    /// Reads raw PCM out of the engine as fast as it is produced. Synthesis runs ahead of
    /// playback once warm, which is exactly what makes her sound continuous.
    private void Pump()
    {
        var engine = _engine;
        var buffer = new byte[8192];

        try
        {
            var stream = engine?.StandardOutput.BaseStream;

            while (stream is not null && !_disposed)
            {
                var read = stream.Read(buffer, 0, buffer.Length);
                if (read <= 0) break;

                _lastAudio = DateTime.UtcNow;
                _awaitingAudio = false;
                _buffer?.AddSamples(buffer, 0, read);
            }
        }
        catch (Exception ex) when (!_disposed)
        {
            Log.Error("her voice stopped reading", ex);
            Trouble?.Invoke("Her voice engine stopped. Getting it back needs a restart.");
        }
    }

    /// Called with the audio at the moment the clock hands it on, so the mouth is in step
    /// with what is heard rather than with what has been generated.
    private void OnAudioPlayed(ReadOnlySpan<byte> pcm)
    {
        /* Tee to any face that asked for her voice, before the visemes are read, so the two
           leave from the same buffer at the same instant.

           The span **must be copied**: it cannot be captured by an async send, and it is a
           view over a buffer that is about to be reused. Rented rather than allocated,
           because at fifty frames a second the garbage would be constant, and returned the
           moment the handlers are done — which is why the event contract says a handler may
           not hold it. */
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

        /* **Whole viseme frames, carried across chunk boundaries.**

           This once read frames straight out of whatever arrived and discarded the
           remainder, which was fine for as long as *whatever arrived* was reliably bigger
           than a frame — `WaveOut` delivered 80 ms, and a frame is 512 samples.

           `Pacer` replaced the sound card with a 20 ms clock, and 20 ms is smaller than a
           frame at every rate she has ever spoken at: 441 samples at Piper's 22,050 Hz, 480
           at Kokoro's 24,000. The loop below simply stopped executing — her voice played
           perfectly and her mouth never moved once, with nothing thrown and nothing logged.

           Leftovers are kept now, so the reader sees a continuous stream and no chunk size
           can silence her mouth again. `VoiceChecks` asserts the property rather than the
           number, so a future change of clock gets told here rather than by a bug report. */
        var arrived = MemoryMarshal.Cast<byte, short>(pcm);

        if (_carry.Count > 0 || arrived.Length < VisemeReader.FrameSamples)
        {
            foreach (var sample in arrived) _carry.Add(sample);
            if (_carry.Count < VisemeReader.FrameSamples) return;
        }

        var samples = _carry.Count > 0 ? CollectionsMarshal.AsSpan(_carry) : arrived;
        var consumed = 0;

        for (var offset = 0; offset + VisemeReader.FrameSamples <= samples.Length; offset += VisemeReader.FrameSamples)
        {
            consumed = offset + VisemeReader.FrameSamples;
            var mouth = reader.Read(samples.Slice(offset, VisemeReader.FrameSamples));

            // The clock pulls continuously and a buffer with nothing in it comes back as
            // silence, so without this she would send a viseme fifty times a second for
            // ever — every one of them saying "mouth shut".
            if (mouth.Shape == _lastShape && Math.Abs(mouth.Openness - _lastOpenness) < 0.02) continue;

            _lastShape = mouth.Shape;
            _lastOpenness = mouth.Openness;
            Viseme?.Invoke(mouth.Openness, mouth.Shape);
        }

        if (_carry.Count > 0) _carry.RemoveRange(0, consumed);
    }

    public void Say(string sentence)
    {
        if (_disposed || string.IsNullOrWhiteSpace(sentence)) return;

        lock (_gate)
        {
            if (_engine is null || _engine.HasExited)
            {
                Trouble?.Invoke("Her voice engine is not running.");
                return;
            }

            _awaitingAudio = true;
            _lastAudio = DateTime.UtcNow;

            if (!_speaking)
            {
                _speaking = true;
                Started?.Invoke();
            }

            try
            {
                // One line in, one utterance out. Newlines inside would be two, and control
                // characters would be read as the engine's own vocabulary.
                _engine.StandardInput.WriteLine(Clean(sentence));
                _engine.StandardInput.Flush();
            }
            catch (Exception ex)
            {
                Log.Error("could not hand a sentence to her voice", ex);
                Trouble?.Invoke("Her voice engine stopped accepting text.");
            }
        }
    }

    private static string Clean(string sentence)
    {
        Span<char> room = sentence.Length <= 512 ? stackalloc char[sentence.Length] : new char[sentence.Length];
        var length = 0;

        foreach (var c in sentence)
            room[length++] = char.IsControl(c) ? ' ' : c;

        return new string(room[..length]);
    }

    /// **She stops, rather than finishing quietly to herself.**
    ///
    /// Piper could not be interrupted: a hush there meant reading its remaining audio and
    /// throwing it away, so the machine kept synthesising a sentence nobody would ever hear.
    /// This engine is asked to abandon the utterance, which it can do mid-word because the
    /// callback generating the audio is the same one that checks.
    public void Hush()
    {
        if (_disposed) return;

        lock (_gate)
        {
            _awaitingAudio = false;
            _buffer?.ClearBuffer();
            _carry.Clear();
            _speaking = false;

            try
            {
                if (_engine is { HasExited: false })
                {
                    _engine.StandardInput.WriteLine(Control + "hush");
                    _engine.StandardInput.Flush();
                }
            }
            catch (Exception ex) { Log.Debug($"hush: {ex.Message}"); }
        }

        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    /// The engine marks nothing, so the end of an utterance is quiet: the playback buffer
    /// empty and nothing new arriving for a moment.
    private void CheckSettled()
    {
        if (!_speaking || _disposed) return;

        // Synthesis takes a moment to produce its first sample, and during that moment the
        // buffer is empty and nothing has arrived — which looks exactly like having
        // finished. She would go idle and then talk. Nothing is over until it has begun.
        if (_awaitingAudio) return;

        var quiet = DateTime.UtcNow - _lastAudio > Settled;
        var drained = (_buffer?.BufferedBytes ?? 0) == 0;
        if (!quiet || !drained) return;

        lock (_gate)
        {
            if (!_speaking) return;
            _speaking = false;
        }

        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    private void StopEngine()
    {
        _watchdog?.Dispose();
        _watchdog = null;

        try { _pacer?.Dispose(); } catch (Exception ex) { Log.Debug($"voice stop: {ex.Message}"); }
        _pacer = null;
        _buffer = null;

        if (_engine is not null)
        {
            try
            {
                if (!_engine.HasExited)
                {
                    _engine.StandardInput.Close();
                    if (!_engine.WaitForExit(1200)) _engine.Kill(entireProcessTree: true);
                }
            }
            catch (Exception ex) { Log.Debug($"voice engine teardown: {ex.Message}"); }

            _engine.Dispose();
            _engine = null;
        }

        _pump = null;
        _speaking = false;
        _carry.Clear();
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        lock (_gate) StopEngine();
    }
}
```
