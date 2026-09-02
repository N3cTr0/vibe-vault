---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Music\MusicReport.cs
---

# src\Octavia.Core\Senses\Music\MusicReport.cs

```csharp
namespace Octavia.Senses.Music;

/// What a client says is playing near it.
///
/// **Reported, not measured** — Stage 15 item 3. She used to hear music herself, two ways:
/// a `LoopbackListener` on the server's own sound card, and a second analyser fed from the
/// server's microphone so a speaker across the room was not silence to her. Both were
/// devices, and the server holds none.
///
/// This is the one part of that rule that could not be solved by moving code from the server
/// to the page. **A browser cannot capture loopback at all** — there is no API for "what this
/// machine is playing", and there is not going to be one. So the sender has to be a shell
/// with real audio access: the Windows client, which already has NAudio, and which can push
/// this through the page's socket rather than opening one of its own.
///
/// Until something reports, she simply does not know — which is honest, and better than the
/// previous answer, where a server in a cupboard confidently reported the silence of a room
/// nobody was in.
internal readonly record struct MusicReport(
    bool Playing,
    double Bpm,
    double Confidence,
    string Source,
    DateTimeOffset At)
{
    public static MusicReport Silent => new(false, 0, 0, "", DateTimeOffset.MinValue);

    /// Whether anything has ever been reported. Distinct from *nothing is playing*: one is
    /// "she has no idea", the other is "she knows the room is quiet", and telling a person
    /// those are the same thing is how a broken sensor reads as a quiet house.
    public bool Known => At != DateTimeOffset.MinValue;

    /// How long ago it was said.
    ///
    /// **A stale report is not cleared**, deliberately. A client that closes its laptop lid
    /// has not told her the music stopped; it has stopped telling her anything, and those are
    /// different. Whoever reads this decides how old is too old.
    public TimeSpan Age => Known ? DateTimeOffset.UtcNow - At : TimeSpan.MaxValue;
}
```
