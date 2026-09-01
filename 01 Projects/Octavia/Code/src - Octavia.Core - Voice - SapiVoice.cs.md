---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\SapiVoice.cs
---

# src\Octavia.Core\Voice\SapiVoice.cs

```csharp
using System.Speech.Synthesis;
using Octavia.Core;

namespace Octavia.Voice;

/// Windows' own synthesiser. It reports real phoneme events while speaking, which is the
/// reason the mouth moved to the host in the first place: a browser could only ever have
/// guessed at syllables from a timer.
///
/// It stays after the neural voice arrives, because it is installed on every Windows
/// machine, starts instantly, and needs nothing downloaded — which makes it the fallback
/// that always works, exactly like the plaster bust.
internal sealed class SapiVoice : IVoice
{
    public string EngineName => $"Windows speech ({CurrentVoice})";

    /// SAPI has nothing to report that is not already an exception; the event exists
    /// because the interface has to carry a neural engine's model downloads.
    public event Action<string>? Trouble;

    private readonly SpeechSynthesizer _synth = new();
    private int _queued;
    private bool _disposed;

    /// Mouth openness (0 shut to 1 wide) and the shape it should take, named as a
    /// VRM viseme so a real avatar needs no translation layer at all.
    public event Action<double, string?>? Viseme;
    public event Action? Started;
    public event Action? Finished;

    /// Never raised. `SetOutputToDefaultAudioDevice` hands the PCM straight to the sound
    /// card and there is no seam to tee; getting at it means `SetOutputToAudioStream` and a
    /// rework, which is deliberately out of scope. See ROADMAP.md stage 14 item 3.
    ///
    /// The warning is suppressed rather than worked around, because "declared and never
    /// raised" is exactly the truth here and the alternative is a fake raise site. Paired
    /// with `AudioFormat` returning null, which is how a face learns not to wait for it.
#pragma warning disable CS0067
    public event Action<ReadOnlyMemory<byte>>? Audio;
#pragma warning restore CS0067

    /// Null, and that is the point: a face is *told* this voice cannot be streamed instead
    /// of waiting in silence for frames that were never coming.
    public AudioFormat? AudioFormat => null;

    public bool IsSpeaking => _queued > 0;

    /// There is no stream to tee here, so silencing this engine silences her entirely —
    /// the room she is attending gets captions and no voice. That is the honest outcome:
    /// `audioAvailable` in `hello` already says this voice cannot leave the machine, and
    /// speaking aloud at the desk instead is exactly the fault rooms exist to fix.
    ///
    /// `Volume` is used rather than pausing, so visemes keep arriving and her mouth still
    /// moves in the room she is talking to.
    private bool _aloud = true;

    public bool Aloud
    {
        get => _aloud;
        set
        {
            if (_aloud == value) return;
            _aloud = value;

            try
            {
                _synth.Volume = value ? 100 : 0;
            }
            catch (Exception ex)
            {
                Log.Write($"could not silence the Windows voice: {ex.Message}");
            }
        }
    }

    public SapiVoice(OctaviaConfig config)
    {
        _synth.SetOutputToDefaultAudioDevice();
        _synth.Rate = Math.Clamp(config.VoiceRate, -10, 10);

        SelectVoice(config.VoiceName);

        _synth.VisemeReached += (_, e) => Viseme?.Invoke(Openness(e.Viseme), Shape(e.Viseme));
        _synth.SpeakCompleted += OnCompleted;
    }

    // Trouble is declared but never raised here; silence the unused-event warning.
    private void Unused() => Trouble?.Invoke(string.Empty);

    public IReadOnlyList<string> InstalledVoices() =>
        _synth.GetInstalledVoices()
              .Where(v => v.Enabled)
              .Select(v => v.VoiceInfo.Name)
              .ToList();

    public string CurrentVoice => _synth.Voice?.Name ?? string.Empty;

    public bool SelectVoice(string? name)
    {
        if (string.IsNullOrWhiteSpace(name)) return false;
        try
        {
            _synth.SelectVoice(name);
            return true;
        }
        catch (Exception ex)
        {
            Log.Write($"voice '{name}' unavailable: {ex.Message}");
            return false;
        }
    }

    public void Say(string sentence)
    {
        if (_disposed || string.IsNullOrWhiteSpace(sentence)) return;

        var wasIdle = Interlocked.Increment(ref _queued) == 1;
        if (wasIdle) Started?.Invoke();

        try
        {
            _synth.SpeakAsync(sentence);
        }
        catch (Exception ex)
        {
            Log.Write($"speak failed: {ex.Message}");
            Settle();
        }
    }

    public void Hush()
    {
        if (_disposed) return;
        try
        {
            _synth.SpeakAsyncCancelAll();
        }
        catch (Exception ex)
        {
            Log.Write($"hush failed: {ex.Message}");
        }

        Interlocked.Exchange(ref _queued, 0);
        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    private void OnCompleted(object? sender, SpeakCompletedEventArgs e) => Settle();

    private void Settle()
    {
        var remaining = Interlocked.Decrement(ref _queued);
        if (remaining > 0) return;

        if (remaining < 0)
        {
            // A cancelled utterance completing after Hush already drained the queue.
            Interlocked.CompareExchange(ref _queued, 0, remaining);
            return;
        }

        Viseme?.Invoke(0, null);
        Finished?.Invoke();
    }

    /// SAPI viseme identifiers mapped to how far the jaw should drop for each.
    private static double Openness(int viseme) => viseme switch
    {
        0 => 0.00,   // silence
        1 => 0.50,   // ae, ax, ah
        2 => 0.90,   // aa
        3 => 0.75,   // ao
        4 => 0.50,   // ey, eh, uh
        5 => 0.45,   // er
        6 => 0.30,   // y, iy, ih, ix
        7 => 0.25,   // w, uw
        8 => 0.65,   // ow
        9 => 0.85,   // aw
        10 => 0.60,  // oy
        11 => 0.85,  // ay
        12 => 0.35,  // h
        13 => 0.30,  // r
        14 => 0.35,  // l
        15 => 0.20,  // s, z
        16 => 0.25,  // sh, ch, jh, zh
        17 => 0.20,  // th, dh
        18 => 0.20,  // f, v
        19 => 0.25,  // d, t, n
        20 => 0.30,  // k, g, ng
        21 => 0.05,  // p, b, m
        _ => 0.25
    };

    /// The same SAPI identifiers, mapped to the five VRM mouth shapes. One number said
    /// how far the jaw dropped; "aa" and "ou" are the same drop with different mouths,
    /// and that difference is most of what makes speech look like speech.
    internal static string? Shape(int viseme) => viseme switch
    {
        0 or 21 => null,              // silence, and p/b/m — a closed mouth
        2 or 1 or 9 or 11 or 12 or 20 => "aa",
        3 or 8 or 10 => "oh",
        7 or 13 or 16 => "ou",
        4 or 5 or 14 => "ee",
        6 or 15 or 17 or 18 or 19 => "ih",
        _ => "aa"
    };

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        try
        {
            _synth.SpeakAsyncCancelAll();
            _synth.Dispose();
        }
        catch (Exception ex)
        {
            Log.Write($"voice dispose failed: {ex.Message}");
        }
    }
}
```
