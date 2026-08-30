---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Audio\VisemeReader.cs
---

# src\Octavia.App\Audio\VisemeReader.cs

```csharp
namespace Octavia.Audio;

/// <param name="Openness">0 shut to 1 wide.</param>
/// <param name="Shape">A VRM viseme name, or null for a closed mouth.</param>
internal readonly record struct Mouth(double Openness, string? Shape);

/// Lip sync read out of the audio itself.
///
/// SAPI handed us phoneme events for free; a neural engine hands us a waveform and
/// nothing else. Rather than tie the mouth to one engine's alignment data, the mouth is
/// derived from the sound she is actually making — which works for **any** voice we ever
/// swap in, and will work for singing along to music in Stage 7.
///
/// **This is an approximation, and worth saying so.** It is not phoneme recognition: it
/// reads loudness for how far the jaw drops, and the balance of energy across three
/// bands for what the lips are doing. Vowels carry most of a mouth's visible movement
/// and are exactly what those bands separate well, so it reads convincingly at
/// conversational speed. Consonants it mostly gets out of the way for.
internal sealed class VisemeReader
{
    /// ~23 ms at 22 kHz. Long enough for the low band to have something to say, short
    /// enough that a mouth shape lands on the syllable that caused it.
    public const int FrameSamples = 512;

    private readonly int _sampleRate;
    private readonly double[] _window = Fft.Hann(FrameSamples);
    private readonly double[] _real = new double[FrameSamples];
    private readonly double[] _imaginary = new double[FrameSamples];

    /// Loudness is relative: a quiet voice should still open her mouth. The peak decays
    /// so one shout does not leave her mumbling for the rest of the conversation.
    private double _peak = 0.02;

    /// Where this voice's front/back measure normally sits. Every engine and every voice
    /// has its own spectral balance, so fixed thresholds tuned on one of them make
    /// another mumble in a single shape. Tracking the centre keeps the mouth moving
    /// through its whole range whoever is speaking.
    private double _frontCentre = 0.22;

    /// The front/back measure behind the last shape, for tuning it against real audio.
    /// The thresholds below are only defensible if you can see what they are cutting.
    public double LastFront { get; private set; }

    public VisemeReader(int sampleRate) => _sampleRate = sampleRate;

    public void Reset() => _peak = 0.02;

    public Mouth Read(ReadOnlySpan<short> frame)
    {
        if (frame.Length < FrameSamples) return new Mouth(0, null);

        double sum = 0;
        for (var i = 0; i < FrameSamples; i++)
        {
            var sample = frame[i] / 32768.0;
            sum += sample * sample;
            _real[i] = sample * _window[i];
            _imaginary[i] = 0;
        }

        var rms = Math.Sqrt(sum / FrameSamples);
        _peak = Math.Max(rms, _peak * 0.9995);

        // Below the noise floor she is between words, and her mouth should be shut.
        if (rms < 0.004) return new Mouth(0, null);

        Fft.Transform(_real, _imaginary);

        // Three bands, chosen around where the first two formants live. Their *balance*
        // is what separates a wide "ee" from a rounded "ou"; absolute level is loudness,
        // which the jaw already has.
        var low = Energy(250, 900);      // F1 — how open the jaw is
        var mid = Energy(900, 1800);
        var high = Energy(1800, 3200);   // F2 — how front and spread the lips are
        var total = low + mid + high;
        if (total <= 1e-12) return new Mouth(0, null);

        var front = (mid * 0.4 + high) / total;   // 0 back and rounded, 1 front and spread
        var openness = Math.Clamp(Math.Pow(rms / Math.Max(_peak, 1e-6), 0.6), 0, 1);

        _frontCentre = Math.Clamp(_frontCentre * 0.99 + front * 0.01, 0.05, 0.6);

        LastFront = front;
        return new Mouth(openness, Shape(front / _frontCentre, openness));
    }

    /// The five VRM mouths on two axes: how open, and how front. `front` arrives here
    /// already divided by this voice's centre, so 1.0 means "typical for her" and the
    /// boundaries are multiples of that rather than absolute numbers.
    ///
    /// The boundaries were set from the measured distribution of real synthesised
    /// speech — a deliberately *distributional* choice, not a phonetic one. It does not
    /// claim to know which vowel she said; it guarantees her mouth moves through its
    /// whole range instead of settling into one shape, which is what reads as talking.
    /// `EarsTest -- mouth <wav>` prints the timeline these came from.
    private static string Shape(double front, double openness) => front switch
    {
        > 1.85 => openness > 0.45 ? "ee" : "ih",
        > 0.55 => openness > 0.55 ? "aa" : "ih",
        _ => openness > 0.60 ? "oh" : "ou"
    };

    /// Bins are weighted by frequency to undo the natural downward tilt of speech —
    /// roughly 6 dB per octave. Without that the low band wins every comparison and
    /// every vowel reads as a rounded "oh", whatever she actually said.
    private double Energy(double fromHz, double toHz)
    {
        var first = (int)(fromHz * FrameSamples / _sampleRate);
        var last = Math.Min((int)(toHz * FrameSamples / _sampleRate), FrameSamples / 2 - 1);

        double energy = 0;
        for (var bin = Math.Max(first, 1); bin <= last; bin++)
        {
            var power = _real[bin] * _real[bin] + _imaginary[bin] * _imaginary[bin];
            energy += power * bin;
        }

        return energy;
    }
}
```
