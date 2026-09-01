---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Music\MusicWatcher.cs
---

# src\Octavia.Core\Senses\Music\MusicWatcher.cs

```csharp
using Octavia.Core;

namespace Octavia.Senses.Music;

/// The sense: a loopback listener feeding an analyser, and the small amount of judgement
/// about when the face is worth telling.
///
/// Everything here is reflex-speed and local. No model is called, which is what makes it
/// affordable to leave running — see the standing constraints in ROADMAP.md.
internal sealed class MusicWatcher : IDisposable
{
    private readonly LoopbackListener _listener = new();
    /// Which ear this is — "output" for the loopback, "room" for the microphone. Two
    /// watchers write the same lines to the same log, and a line that cannot say which
    /// one it came from is worse than no line.
    public string Name { get; init; } = "output";

    private MusicAnalyzer? _analyzer;

    private MusicState _last;
    private DateTime _lastSent = DateTime.MinValue;
    private bool _wanted;      // whether she is supposed to be listening at all
    private bool _reopening;   // a device change is already being followed

    /// Raised when the face should be told something new. Arrives on a capture thread.
    public event Action<MusicState>? Changed;

    /// Raised on the beat, so a renderer can move on it rather than guessing.
    public event Action? Beat;

    /// Set while Octavia is speaking. Her voice reaches the loopback like anything else.
    public bool Hold { get; set; }

    public bool IsRunning => _listener.IsRunning;
    public MusicState State => _last;
    public string DeviceName => _listener.DeviceName;

    /// Which render endpoint to tap. Empty follows the Windows default.
    public string? Device { get => _listener.Device; set => _listener.Device = value; }
    public int SampleRate => _listener.SampleRate;

    /// How completely, and how faithfully, the audio path is delivering — see
    /// `LoopbackListener.FramesSeen` and `Crest`.
    public (long Frames, long Gaps, long Silent, double Crest) Delivery =>
        (_listener.FramesSeen, _listener.Gaps, _listener.SilentBuffers, _listener.Crest);

    /// Peak and RMS behind the crest factor, for when the ratio alone is ambiguous.
    public (double Peak, double Rms) Levels => (_listener.Peak, _listener.Rms);

    /// Analyse audio handed in from elsewhere instead of opening a loopback.
    ///
    /// **This is how she hears a room.** Loopback is what *this computer* is playing; a
    /// speaker across the room — another PC, a phone, a hi-fi — never touches it and only
    /// ever reaches the microphone. For something that lives in a room rather than on a
    /// desktop that was the wrong way round, and it looked exactly like the beat detection
    /// being broken. See ROADMAP.md stage 11a.
    ///
    /// The same analyser either way, so everything measured about tempo still holds. What
    /// does *not* hold is the crest factor: a boom mic in a reverberant room with gain
    /// control on it will never deliver the dynamics a loopback does, so expect this
    /// source to be less certain and treat its confidence accordingly.
    public void StartFromFrames(int sampleRate)
    {
        _wanted = true;
        _external = true;
        _analyzer = new MusicAnalyzer(sampleRate);
    }

    /// Frames from that other source, at the rate given to StartFromFrames. Called on
    /// whatever thread captured them, exactly as the loopback's own callback is.
    public void Push(float[] samples, int count)
    {
        if (_external) OnFrames(samples, count);
    }

    private bool _external;

    public async Task<bool> StartAsync()
    {
        if (_listener.IsRunning) return true;

        _wanted = true;
        _listener.Frames += OnFrames;
        _listener.Stopped += OnListenerStopped;

        if (!await _listener.StartAsync())
        {
            _listener.Frames -= OnFrames;
            _listener.Stopped -= OnListenerStopped;
            return false;
        }

        _analyzer = new MusicAnalyzer(_listener.SampleRate);
        return true;
    }

    /// Plugging headphones in ends the capture rather than moving it, because a loopback
    /// cannot follow the default device. Without this she would simply stop hearing music
    /// for the rest of the session, with nothing on screen to say why.
    private void OnListenerStopped()
    {
        if (!_wanted || _reopening) return;
        _reopening = true;

        Task.Run(async () =>
        {
            try
            {
                // Long enough for Windows to finish settling on the new endpoint.
                await Task.Delay(TimeSpan.FromSeconds(2));
                _listener.Stop();

                if (_wanted && await _listener.StartAsync())
                {
                    _analyzer = new MusicAnalyzer(_listener.SampleRate);
                    Log.Write($"music: following the output to '{_listener.DeviceName}'");
                }
            }
            finally
            {
                _reopening = false;
            }
        }).Forget("reopening the loopback after the output device changed");
    }

    public void Stop()
    {
        _wanted = false;

        if (_external)
        {
            _external = false;
            _analyzer = null;
            if (_last.Playing || _last.Bpm > 0) { _last = default; Changed?.Invoke(_last); }
            return;
        }

        if (!_listener.IsRunning) return;

        _listener.Frames -= OnFrames;
        _listener.Stopped -= OnListenerStopped;
        _listener.Stop();
        _analyzer = null;

        if (_last.Playing || _last.Bpm > 0)
        {
            _last = default;
            Changed?.Invoke(_last);
        }
    }

    private void OnFrames(float[] samples, int count)
    {
        var analyzer = _analyzer;
        if (analyzer is null) return;

        int beats;
        MusicState state;

        try
        {
            beats = analyzer.Push(samples.AsSpan(0, count), Hold);
            state = analyzer.State;
        }
        catch (Exception ex)
        {
            Log.Error("music analysis failed", ex);
            return;
        }

        // A beat is the one thing that is worthless late, so it goes out immediately and
        // is never coalesced. Everything else can wait for the next update.
        for (var i = 0; i < beats; i++) Beat?.Invoke();

        // The transition is logged where `_last` is actually updated, further down —
        // logging it here re-reported the same change on every frame until the throttle
        // below finally let a send through and moved `_last` on. Five identical lines in
        // one second was that, not two sources talking over each other.
        var wasPlaying = _last.Playing;

        // Energy at roughly twelve a second, and only when it moved: the same reasoning
        // as the microphone level, whose stream this one sits beside.
        var now = DateTime.UtcNow;
        var moved = Math.Abs(state.Energy - _last.Energy) > 0.04
                 || Math.Abs(state.Bpm - _last.Bpm) > 1.0
                 || state.Playing != _last.Playing;

        if (!moved || (now - _lastSent).TotalMilliseconds < 80) return;

        _lastSent = now;
        _last = state;

        if (state.Playing != wasPlaying)
        {
            Log.Write(state.Playing
                ? $"music ({Name}): {state.Bpm:0} bpm (confidence {state.Confidence:0.00})"
                : $"music ({Name}): stopped");
        }

        Changed?.Invoke(state);
    }

    public void Dispose()
    {
        Stop();
        _listener.Dispose();
    }
}
```
