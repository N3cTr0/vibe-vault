---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Voice\IVoice.cs
---

# src\Octavia.App\Voice\IVoice.cs

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

    /// Something the user should know — a model downloading, an engine that would not
    /// start. Reported rather than thrown, the same way the ears do it.
    event Action<string>? Trouble;

    bool IsSpeaking { get; }

    /// What this engine calls itself, for the face and the diagnostics report.
    string EngineName { get; }

    IReadOnlyList<string> InstalledVoices();
    string CurrentVoice { get; }
    bool SelectVoice(string? name);

    void Say(string sentence);
    void Hush();
}
```
