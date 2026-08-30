---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Music\MusicAnalyzer.cs
---

# src\Octavia.App\Senses\Music\MusicAnalyzer.cs

```csharp
using Octavia.Audio;

namespace Octavia.Senses.Music;

/// <param name="Playing">Whether this is music rather than speech, a notification or silence.</param>
/// <param name="Bpm">0 until a tempo has been found.</param>
/// <param name="Energy">0-1 loudness, relative to how loud this material has been.</param>
/// <param name="Confidence">How strongly the onset envelope repeats at the chosen tempo.</param>
internal readonly record struct MusicState(bool Playing, double Bpm, double Energy, double Confidence);

/// Beat detection, with no model and no network.
///
/// The chain is the standard one: a spectral-flux onset envelope, autocorrelated to find
/// a tempo, then matched against a pulse train to find where the beat actually falls. It
/// is deliberately the same shape as the mouth in Stage 6 — an FFT and some bookkeeping —
/// because "dancing is free" is a constraint, not an aspiration. See ROADMAP.md.
///
/// Pure and device-free on purpose: `LoopbackListener` supplies the samples, so this can
/// be tested against a synthesized click track at a known tempo rather than by playing
/// something and watching her.
internal sealed class MusicAnalyzer
{
    public const int Window = 1024;
    public const int Hop = 512;

    /// How much onset envelope to keep. Long enough to hold four bars of anything slower
    /// than a dirge, short enough to follow a track change. Held in *seconds* rather than
    /// frames because the loopback runs at whatever the sound card does, and a constant
    /// number of hops would quietly mean sixteen seconds of memory at 16 kHz.
    private const double HistorySeconds = 6.0;

    /// Tempo range. Below 60 the autocorrelation starts finding bar lines instead of
    /// beats; above 190 it starts finding the off-beats of something half that speed.
    private const double MinBpm = 60, MaxBpm = 190;

    /// How nearly as periodic twice the tempo has to be before it is preferred, as a
    /// share of the winning lag's autocorrelation.
    ///
    /// Measured rather than picked. On material with a backbeat, where doubling is the
    /// right answer, the half lag scores 0.63–0.67; where it is the wrong answer the two
    /// are anti-correlated and it scores about −0.03. The gap is wide and it has a sign
    /// to it, which is what makes a single threshold defensible.
    private const double OctaveShare = 0.45;

    private readonly double _frameRate;
    private readonly double[] _window = Fft.Hann(Window);
    private readonly double[] _real = new double[Window];
    private readonly double[] _imaginary = new double[Window];
    private readonly double[] _previous;
    private readonly int _history;
    private readonly double[] _flux;
    private readonly double[] _loud;

    /// Autocorrelation per candidate lag, kept so the octave check below can ask what
    /// twice the tempo looked like. Reused rather than allocated six times a second.
    private readonly double[] _acf;
    private readonly float[] _pending = new float[Window];

    private readonly int _lowBin, _highBin;
    private int _filled;      // samples held in _pending
    private long _hops;       // hops processed since construction
    private int _cursor;      // write position in the ring buffers

    private double _peak = 1e-4;
    private double _energy;
    private double _period;   // hops per beat, 0 until found
    private double _confidence;
    private double _nextBeat = double.MaxValue;  // in hops
    private double _sure;     // seconds of agreement, for the playing decision
    private long _lastSearch;   // never negative: _hops - _lastSearch would overflow
    private double _carried;    // fractional hops left over from the last coasted buffer

    public MusicAnalyzer(int sampleRate)
    {
        _frameRate = (double)sampleRate / Hop;
        _previous = new double[Window / 2];
        _history = Math.Clamp((int)(HistorySeconds * _frameRate), 256, 1024);
        _flux = new double[_history];
        _loud = new double[_history];
        _acf = new double[(int)(_frameRate * 60.0 / MinBpm) + 2];

        // Below 40 Hz is rumble and DC; above 5 kHz is mostly cymbals and hiss, which
        // move constantly and swamp the flux with things that are not onsets.
        _lowBin = Math.Max(1, (int)(40.0 * Window / sampleRate));
        _highBin = Math.Min(Window / 2 - 1, (int)(5000.0 * Window / sampleRate));
    }

    public MusicState State => new(_sure >= 1.0, Bpm, _energy, _confidence);

    public double Bpm => _period > 0 ? Math.Round(60.0 * _frameRate / _period, 1) : 0;

    /// How much of the recent past was loud enough to be something rather than nothing.
    /// Speech is full of gaps; music mostly is not, and this is the cheapest way to tell
    /// them apart that does not involve a classifier.
    public double Continuity { get; private set; }

    public void Reset()
    {
        Array.Clear(_flux);
        Array.Clear(_loud);
        Array.Clear(_previous);
        _filled = 0; _hops = 0; _cursor = 0;
        _peak = 1e-4; _energy = 0; _period = 0; _confidence = 0;
        _nextBeat = double.MaxValue; _sure = 0; _lastSearch = 0;
        Continuity = 0;
    }

    /// Feeds mono samples in and returns how many beats fell inside them.
    ///
    /// `hold` is set while Octavia is speaking. Her own voice comes back through the
    /// loopback like anything else, so analysing it would let her hear herself and call
    /// it music. Instead the tempo already found keeps running — she stays in time with
    /// a track she is talking over — and the decision is frozen until she stops.
    public int Push(ReadOnlySpan<float> mono, bool hold = false)
    {
        if (hold) return Coast(mono.Length);

        var beats = 0;
        var offset = 0;

        while (offset < mono.Length)
        {
            var take = Math.Min(Window - _filled, mono.Length - offset);
            mono.Slice(offset, take).CopyTo(_pending.AsSpan(_filled));
            _filled += take;
            offset += take;

            if (_filled < Window) break;

            beats += Analyse();

            // Overlap by half a window: the onset envelope needs the resolution, and a
            // beat landing 20 ms late is visible when something is moving to it.
            Array.Copy(_pending, Hop, _pending, 0, Window - Hop);
            _filled = Window - Hop;
        }

        return beats;
    }

    /// Advances the beat clock without listening. Keeps her in time while she talks.
    private int Coast(int samples)
    {
        if (_period <= 0 || _nextBeat == double.MaxValue) return 0;

        // A buffer is not a whole number of hops, so the remainder has to be carried.
        // Dropping it stops the clock entirely: ten-millisecond buffers are less than
        // one hop each, and every advance would truncate to nothing.
        _carried += samples / (double)Hop;
        var whole = (long)_carried;
        _carried -= whole;
        _hops += whole;
        _energy *= Math.Pow(0.98, whole);

        var beats = 0;
        while (_hops >= _nextBeat) { _nextBeat += _period; beats++; }
        return beats;
    }

    private int Analyse()
    {
        double sum = 0;
        for (var i = 0; i < Window; i++)
        {
            var sample = _pending[i];
            sum += sample * sample;
            _real[i] = sample * _window[i];
            _imaginary[i] = 0;
        }

        var rms = Math.Sqrt(sum / Window);
        _peak = Math.Max(rms, _peak * 0.99954);          // ~30 s to forget a loud passage
        _energy = Math.Clamp(Math.Pow(rms / Math.Max(_peak, 1e-6), 0.55), 0, 1);

        Fft.Transform(_real, _imaginary);

        // Spectral flux: only *rises* count. A note starting is an onset; a note ending
        // is not, and counting both turns every beat into two.
        double flux = 0;
        for (var bin = _lowBin; bin <= _highBin; bin++)
        {
            var magnitude = Math.Sqrt(_real[bin] * _real[bin] + _imaginary[bin] * _imaginary[bin]);
            var rise = magnitude - _previous[bin];
            if (rise > 0) flux += rise;
            _previous[bin] = magnitude;
        }

        _flux[_cursor] = flux;
        _loud[_cursor] = rms;
        _cursor = (_cursor + 1) % _history;
        _hops++;

        // Re-deciding the tempo every hop would be waste and would make it jitter; six
        // times a second is far faster than anyone changes track.
        if (_hops - _lastSearch >= 16 && _hops >= _history / 2)
        {
            _lastSearch = _hops;
            Search();
        }

        Decide();

        var beats = 0;
        while (_nextBeat != double.MaxValue && _hops >= _nextBeat)
        {
            _nextBeat += _period;
            beats++;
        }

        return beats;
    }

    /// Reads the ring buffer oldest-first into a plain array, mean removed.
    private double[] Envelope()
    {
        var count = (int)Math.Min(_hops, _history);
        var envelope = new double[count];
        for (var i = 0; i < count; i++)
            envelope[i] = _flux[(_cursor - count + i + _history * 2) % _history];

        var mean = envelope.Average();
        for (var i = 0; i < count; i++) envelope[i] -= mean;
        return envelope;
    }

    private void Search()
    {
        var envelope = Envelope();
        var minLag = (int)(_frameRate * 60.0 / MaxBpm);
        var maxLag = (int)(_frameRate * 60.0 / MinBpm);
        if (maxLag >= envelope.Length - 8) maxLag = envelope.Length - 8;
        if (minLag < 2 || maxLag <= minLag) return;

        var raw = _acf;
        double best = 0, total = 0;
        var bestLag = 0;
        var counted = 0;

        for (var lag = minLag; lag <= maxLag; lag++)
        {
            double sum = 0;
            for (var i = lag; i < envelope.Length; i++) sum += envelope[i] * envelope[i - lag];
            sum /= envelope.Length - lag;
            raw[lag - minLag] = sum;

            // Tempo is perceived logarithmically and 120 is the middle of it. Without
            // this the autocorrelation happily returns half or double the tempo a person
            // would tap, since both are genuinely periods of the music.
            var bpm = 60.0 * _frameRate / lag;
            var weighted = sum * Math.Exp(-0.5 * Math.Pow(Math.Log(bpm / 120.0) / 0.65, 2));

            total += Math.Abs(weighted);
            counted++;

            if (weighted > best) { best = weighted; bestLag = lag; }
        }

        var mean = counted > 0 ? total / counted : 0;
        _confidence = mean > 1e-12 ? Math.Clamp(best / mean / 6.0, 0, 1) : 0;

        if (bestLag <= 0 || _confidence < 0.25) return;

        var (period, strength) = Peak(raw, bestLag, minLag, maxLag);

        // Almost all popular music puts a different sound on two and four, which makes
        // the *bar* more strongly periodic than the beat — so the strongest lag is often
        // half the tempo a person would tap. If twice as fast is nearly as periodic and
        // still a plausible tempo, it is the one to move to.
        if (bestLag / 2 >= minLag + 1)
        {
            var (halfPeriod, halfStrength) = Peak(raw, bestLag / 2, minLag, maxLag);
            if (strength > 0 && halfStrength >= strength * OctaveShare) period = halfPeriod;
        }

        // Ease towards a new tempo rather than snapping: a single confused window
        // should not make her stumble.
        _period = _period > 0 && Math.Abs(period - _period) < _period * 0.25
            ? _period * 0.7 + period * 0.3
            : period;

        Phase(envelope);
    }

    /// The local peak nearest an integer lag, interpolated.
    ///
    /// A tempo is almost never a whole number of hops — 150 bpm is exactly 37.5 of them —
    /// and correlating at the nearest integer both understates the peak and rounds the
    /// answer. It understates it *unevenly*, which is the real damage: a bar line that
    /// happens to land on a whole number outscores the beat that does not, and she ends
    /// up dancing at half speed.
    private static (double Lag, double Value) Peak(double[] raw, int target, int minLag, int maxLag)
    {
        var from = Math.Max(minLag + 1, target - 1);
        var to = Math.Min(maxLag - 1, target + 1);
        var at = from;
        for (var lag = from; lag <= to; lag++)
            if (raw[lag - minLag] > raw[at - minLag]) at = lag;

        var i = at - minLag;
        double a = raw[i - 1], b = raw[i], c = raw[i + 1];
        var curvature = a - 2 * b + c;
        var shift = Math.Abs(curvature) > 1e-12 ? Math.Clamp(0.5 * (a - c) / curvature, -0.5, 0.5) : 0;

        return (at + shift, b - 0.25 * (a - c) * shift);
    }

    /// Where in the bar we are. Scores a pulse train at every offset within one period
    /// and keeps the best — the beat is wherever the onsets actually pile up.
    private void Phase(double[] envelope)
    {
        var period = (int)Math.Round(_period);
        if (period < 2) return;

        double best = double.NegativeInfinity;
        var bestOffset = 0;

        for (var offset = 0; offset < period; offset++)
        {
            double score = 0;
            for (var i = envelope.Length - 1 - offset; i >= 0; i -= period) score += envelope[i];
            if (score > best) { best = score; bestOffset = offset; }
        }

        // bestOffset counts back from the newest hop, so the next beat is one period on.
        var last = _hops - bestOffset;
        _nextBeat = last + _period;
    }

    /// Music or not. Three things have to agree, and they were chosen to disagree on
    /// speech: it is loud enough to be something, it does not stop every few words, and
    /// its onsets repeat at a steady rate.
    private void Decide()
    {
        var count = (int)Math.Min(_hops, _history);
        var floor = Math.Max(_peak * 0.06, 0.0016);
        var loud = 0;
        for (var i = 0; i < count; i++) if (_loud[i] > floor) loud++;
        Continuity = count > 0 ? loud / (double)count : 0;

        var agrees = _confidence > 0.42 && Continuity > 0.72 && _energy > 0.10 && _period > 0;

        // Slow to start and slower to stop: a drum fill should not undress her, and a
        // single loud advert should not put headphones on.
        _sure = Math.Clamp(_sure + (agrees ? 1.0 : -0.6) / _frameRate, 0, 2.5);

        if (_sure < 1.0 && !agrees && _confidence < 0.2)
        {
            _period = 0;
            _nextBeat = double.MaxValue;
        }
    }
}
```
