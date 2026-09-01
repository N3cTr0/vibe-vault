---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Conversation.cs
---

# src\Octavia.Core\Brain\Conversation.cs

```csharp
namespace Octavia.Brain;

internal sealed record Utterance(string Role, string Text)
{
    public const string User = "user";
    public const string Assistant = "assistant";
}

/// The running conversation, kept provider-neutral so both brains share one shape
/// and neither owns the trimming policy.
internal sealed class Conversation
{
    private readonly List<Utterance> _turns = [];
    private readonly int _maxTurns;

    public Conversation(int maxTurns = 40) => _maxTurns = maxTurns;

    public IReadOnlyList<Utterance> Turns => _turns;

    public void Add(string role, string text)
    {
        _turns.Add(new Utterance(role, text));
        // Drop whole exchanges, never half of one: an orphaned assistant turn at the
        // front makes some providers reject the request outright.
        while (_turns.Count > _maxTurns) _turns.RemoveRange(0, 2);
    }

    public void DropLast()
    {
        if (_turns.Count > 0) _turns.RemoveAt(_turns.Count - 1);
    }

    public void Clear() => _turns.Clear();
}
```
