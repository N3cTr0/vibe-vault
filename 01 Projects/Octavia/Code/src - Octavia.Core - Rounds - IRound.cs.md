---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Rounds\IRound.cs
---

# src\Octavia.Core\Rounds\IRound.cs

```csharp
namespace Octavia.Rounds;

/// Something she checks on her own, on a clock, without being asked.
///
/// **This is the seam that makes her able to speak first.** Everything she has ever said
/// until now was a reply — to a held button, a typed line, a wake word — and `OctaviaSession`
/// is built around that: one turn in flight, one room attended, a floor that somebody took.
/// A round is the other thing: work she begins, whose usual and correct outcome is that she
/// says nothing at all.
internal interface IRound
{
    /// For the log and the health panel. A round that never speaks still has to be visible,
    /// or an unfired job looks exactly like a working one.
    string Name { get; }

    /// Null means *nothing worth saying*, which is the normal answer and must stay cheap.
    Task<Finding?> CheckAsync(CancellationToken cancel);
}

/// Something worth interrupting her owner for.
///
/// `Sentence` is what she will say, composed **by the round from its own data** rather than
/// by the model. That is deliberate and it is the whole defence against crying wolf: a model
/// asked hourly whether anything looks concerning will find something concerning. The
/// threshold belongs in the data — a severity the source itself reported — and the model's
/// job starts afterwards, when a person asks a follow-up and the finding is in the history
/// for it to answer from.
///
/// <param name="Sentence">What she says. One or more sentences of ordinary prose.</param>
/// <param name="Trigger">
/// What prompted her, recorded in the room's history in the person's slot so a follow-up
/// question has something to answer from. It is not a lie about who spoke: a round's finding
/// genuinely is an input to her, arriving from somewhere other than a microphone.
/// </param>
internal sealed record Finding(string Sentence, string Trigger);
```
