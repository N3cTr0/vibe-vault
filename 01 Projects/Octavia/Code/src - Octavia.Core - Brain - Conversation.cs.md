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

    public void Clear()
    {
        _turns.Clear();
        _awaiting = null;
    }

    /* ---- a spoken yes, and how long it lasts -------------------------------------- */

    /// A dangerous tool she has asked about and not yet had an answer on.
    ///
    /// The arguments are part of it, not just the name. *"Yes"* to unlocking the back door
    /// is not consent to unlock the front one, and a model that reached for the wrong entity
    /// on the second attempt would otherwise be handed a confirmation that was never about
    /// it.
    internal sealed record Consent(string Tool, string Arguments);

    private Consent? _awaiting;

    /// Remembers what she just asked about, for exactly one turn.
    internal void AwaitYes(string tool, string arguments) => _awaiting = new Consent(tool, arguments);

    /// Reads the pending question and forgets it in the same breath.
    ///
    /// **One turn, and only the next one.** Called at the top of every turn, so a question
    /// she asked is answerable by the very next thing said and by nothing after it. That is
    /// deliberately the tightest rule available: a consent that survived several turns would
    /// let a *"yes"* about something else entirely unlock a door, and *"say yes to the next
    /// thing she asks"* is a sentence a television can produce.
    internal Consent? TakeConsent()
    {
        var carried = _awaiting;
        _awaiting = null;
        return carried;
    }

    /// Whether this utterance grants that pending question, for this exact call.
    ///
    /// **Biased towards refusing.** A yes misread as a no costs one repeated question; a no
    /// misread as a yes costs whatever the tool does. So agreement must be present *and*
    /// disagreement must be absent — "yes, but not the garage" is not consent, and neither
    /// is "no, go ahead" however unlikely a person is to say it.
    internal static bool Grants(Consent? pending, string tool, string arguments, string said)
    {
        if (pending is null) return false;
        if (pending.Tool != tool || pending.Arguments != arguments) return false;

        var words = Words(said);
        if (words.Any(Refuses)) return false;
        return words.Any(Agrees) || Phrase(said);
    }

    private static IEnumerable<string> Words(string said) =>
        said.ToLowerInvariant().Split(
            [' ', ',', '.', '!', '?', ';', ':', '\n', '\r', '\t', '"', '\''],
            StringSplitOptions.RemoveEmptyEntries);

    private static bool Agrees(string word) => word is
        "yes" or "yeah" or "yep" or "yup" or "sure" or "ok" or "okay" or "please" or
        "confirm" or "confirmed" or "affirmative" or "correct" or "proceed";

    private static bool Refuses(string word) => word is
        "no" or "nope" or "not" or "don't" or "dont" or "never" or "cancel" or "stop" or
        "wait" or "hold" or "nevermind" or "actually" or "instead" or "but";

    /// The two-word agreements a single word cannot see.
    private static bool Phrase(string said)
    {
        var flat = said.ToLowerInvariant();
        return flat.Contains("go ahead") || flat.Contains("do it") || flat.Contains("go on");
    }
}
```
