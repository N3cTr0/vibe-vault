---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\ISpeechRecognizer.cs
---

# src\Octavia.Core\Senses\ISpeechRecognizer.cs

```csharp
namespace Octavia.Senses;

internal interface ISpeechRecognizer : IDisposable
{
    /// Final transcript plus the engine's confidence in it.
    event Action<string, float>? Recognized;

    /// Partial transcript, good enough to show but not to act on.
    event Action<string>? Hypothesised;

    /// Something is wrong that the user can fix — most often a microphone that is
    /// open but delivering pure silence. Raised at most once per listening session.
    event Action<string>? Trouble;

    string EngineName { get; }
    bool IsListening { get; }

    void Start();
    void Stop();

    /// Held while Octavia is speaking so she doesn't transcribe her own voice.
    void Mute();
    void Unmute();
}
```
