---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Ears\AwakeWindow.cs
---

# src\Octavia.Core\Senses\Ears\AwakeWindow.cs

```csharp
namespace Octavia.Senses;

/// How long she keeps listening after being addressed.
///
/// **Its own type because getting it wrong is silent.** This rule lived as three fields and
/// two methods on `WhisperRecognizer`, and in v0.52.0 it dropped the question that followed
/// the wake phrase: the window was measured from the wake, a local brain took thirty-six
/// seconds to answer, and it expired while she was still thinking. Nothing threw. The only
/// evidence was a log line blaming the wake word for a fault that was not its own.
///
/// A rule that fails that quietly is worth being able to assert on, and asserting on it
/// should not cost 1.6 GB of speech model to construct.
internal sealed class AwakeWindow
{
    private DateTime _until = DateTime.MinValue;

    /// How long she stays awake past the last thing that kept her there.
    public TimeSpan For { get; set; } = TimeSpan.FromSeconds(25);

    /// Held open regardless of the clock — she is mid-answer.
    public bool Held { get; private set; }

    public bool IsOpen => Held || DateTime.UtcNow < _until;

    /// Something kept her awake: the phrase, or a sentence she just finished saying.
    public void Wake() => _until = DateTime.UtcNow + For;

    /// **Releasing the hold starts the clock rather than stopping it.** She has just finished
    /// answering, which is the moment somebody is most likely to say the next thing — closing
    /// the door on that instant is the bug this type exists to remember.
    public void Hold(bool answering)
    {
        Held = answering;
        if (!answering) Wake();
    }

    /// Shut, and not re-armed. For leaving a room rather than finishing a turn.
    public void Close()
    {
        Held = false;
        _until = DateTime.MinValue;
    }
}
```
