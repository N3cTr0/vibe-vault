---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\IVoice.cs
---

# src\Octavia.Core\Voice\IVoice.cs

```csharp
namespace Octavia.Voice;

/// What she speaks with. Windows' own synthesiser and a neural engine sit behind this,
/// exactly as `ISpeechRecognizer` and `IBrain` hide Whisper and Claude — a voice is a
/// swap, and `OctaviaSession` never learns which one it got.
internal interface IVoice : IDisposable
{
    /// Mouth openness (0 shut to 1 wide) and the shape it should take, named as a VRM
    /// viseme. See PROTOCOL.md.
    event Action<double, string?>? Viseme;

    /// She has started an utterance, and has finished everything queued.
    event Action? Started;
    event Action? Finished;

    /// The audio as it reaches the sound card. Raw PCM in the format `AudioFormat`
    /// advertises, so a face in another room can hear her.
    ///
    /// Teed at the point it goes out rather than anywhere earlier, because that is the one
    /// place where being in step with the visemes is guaranteed — they are read from the
    /// same buffer at the same moment. Anywhere upstream and the two drift.
    ///
    /// The handler must not hold the memory: it is pooled and returned as soon as the
    /// event returns.
    event Action<ReadOnlyMemory<byte>>? Audio;

    /// What this engine can stream, or null if it cannot stream at all.
    ///
    /// Null is an answer, not a gap. A face that is *told* the voice cannot be streamed
    /// shows the difference between a limitation and a bug, instead of waiting in silence
    /// for frames that were never coming.
    AudioFormat? AudioFormat { get; }

    /// Something the user should know — a model downloading, an engine that would not
    /// start. Reported rather than thrown, the same way the ears do it.
    event Action<string>? Trouble;

    bool IsSpeaking { get; }

    /// Whether **this machine's speakers** should hear her. True normally; false while she
    /// is attending a room that is not this one.
    ///
    /// Silencing the sound card rather than not speaking is deliberate: the visemes, the
    /// `Audio` tee and the state machine all read from the audio as it is played, so a face
    /// in the other room still gets her voice, in step with her mouth. Stopping short of the
    /// output is the only place that can be cut without taking the rest with it.
    ///
    /// An engine that cannot stream (SAPI) simply goes quiet — there is nowhere for the
    /// sound to go instead, and a phone conversation playing out loud in an empty house is
    /// the thing this exists to stop.
    bool Aloud { get; set; }

    /// What this engine calls itself, for the face and the diagnostics report.
    string EngineName { get; }

    IReadOnlyList<string> InstalledVoices();
    string CurrentVoice { get; }
    bool SelectVoice(string? name);

    void Say(string sentence);
    void Hush();
}

/// How to interpret the bytes on the wire. Announced in `hello`, never assumed: the rate
/// comes from the voice's own config and changes when the voice does, so a face must
/// re-read it on every `hello` rather than caching it once.
/// <param name="Rate">Samples per second.</param>
/// <param name="Bits">Bits per sample.</param>
/// <param name="Channels">1 for mono.</param>
internal readonly record struct AudioFormat(int Rate, int Bits, int Channels);
```
