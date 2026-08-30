---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Brain\IBrain.cs
---

# src\Octavia.App\Brain\IBrain.cs

```csharp
namespace Octavia.Brain;

/// What Octavia thinks with. Claude in life, a small local model during development
/// and — later — as the cheap gate that decides what deserves a Claude call.
internal interface IBrain : IDisposable
{
    /// Shown on her face, e.g. "claude-sonnet-5" or "qwen3:4b (local)".
    string Description { get; }

    /// False when she cannot think at all: no key, no server.
    bool IsReady { get; }

    /// True only for brains whose readiness a pasted API key would fix.
    bool NeedsApiKey { get; }

    void Forget();

    /// Yields the reply one speakable sentence at a time, so the voice can start
    /// before the model has finished writing.
    ///
    /// `now` is what is true at this moment rather than what was said — music playing,
    /// and what the camera saw when she was asked to look. It rides with the current
    /// question and is never added to the history, so it cannot still be claiming there
    /// is music an hour after it stopped. It stays out of the system prompt for the same
    /// reason the cache breakpoint sits there: anything volatile above it voids the cache.
    ///
    /// A brain that cannot use part of it ignores that part. `LocalBrain` has no eyes.
    IAsyncEnumerable<string> RespondAsync(
        string userText, Situation now = default, CancellationToken cancel = default);
}
```
