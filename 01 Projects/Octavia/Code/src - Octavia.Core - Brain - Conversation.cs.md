---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Conversation.cs
---

# src\Octavia.Core\Brain\Conversation.cs

```csharp
using System.Text;
using System.Text.Json;
using Octavia.Core;

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

    /// The call a spoken yes authorises, or null.
    ///
    /// **It returns the call she described, not the one the model writes next**, and that is
    /// the whole of the v0.49.1 fix. Until then this compared the pending arguments against
    /// the model's *second* attempt and ran the call only if they matched — which asks two
    /// independent generations to agree on a JSON object, and they do not:
    ///
    ///     asked:    set_port_power{"port":1,"on":false}
    ///     after yes: set_port_power{"device":"unifi-gateway","port":1,"on":false}
    ///
    /// Same intent, an argument added, refused — so she asked again, and the model added
    /// something again, and the person answered yes for ever. Comparing them as JSON rather
    /// than as text (v0.49.0) fixed the *spelling* half of this and could never have fixed
    /// this half.
    ///
    /// So the model is no longer asked to reproduce anything. What runs is the call that was
    /// **described out loud to the person**, which is also the only call they can be said to
    /// have agreed to — strictly safer than trusting a second generation to be equivalent.
    ///
    /// **Biased towards refusing.** A yes misread as a no costs one repeated question; a no
    /// misread as a yes costs whatever the tool does. So agreement must be present *and*
    /// disagreement absent — "yes, but not the garage" is not consent, and neither is
    /// "no, go ahead" however unlikely a person is to say it.
    internal static Consent? Authorises(Consent? pending, string tool, string said)
    {
        if (pending is null) return null;

        if (pending.Tool != tool)
        {
            Log.Write($"consent was for '{pending.Tool}', not '{tool}'; asking again");
            return null;
        }

        var words = Words(said);
        if (words.Any(Refuses))
        {
            Log.Write($"'{said}' reads as a refusal; leaving {tool} alone");
            return null;
        }

        if (words.Any(Agrees) || Phrase(said)) return pending;

        Log.Write($"'{said}' is neither a yes nor a no; leaving {tool} alone");
        return null;
    }

    /// Two argument objects that mean the same call, whatever order or spacing they arrived in.
    internal static bool SameCall(string a, string b) =>
        string.Equals(a, b, StringComparison.Ordinal) || Canonical(a) == Canonical(b);

    /// A stable text for one JSON value: keys sorted, no whitespace, recursively.
    ///
    /// Unparsable arguments fall back to their trimmed text rather than throwing — a call
    /// whose arguments would not parse is refused elsewhere, and this is not the place to
    /// discover it.
    private static string Canonical(string json)
    {
        try
        {
            using var document = JsonDocument.Parse(json);
            var builder = new StringBuilder();
            Write(document.RootElement, builder);
            return builder.ToString();
        }
        catch (JsonException)
        {
            return json.Trim();
        }
    }

    private static void Write(JsonElement element, StringBuilder builder)
    {
        switch (element.ValueKind)
        {
            case JsonValueKind.Object:
                builder.Append('{');
                var first = true;
                foreach (var property in element.EnumerateObject().OrderBy(p => p.Name, StringComparer.Ordinal))
                {
                    if (!first) builder.Append(',');
                    first = false;
                    builder.Append(JsonSerializer.Serialize(property.Name)).Append(':');
                    Write(property.Value, builder);
                }
                builder.Append('}');
                break;

            case JsonValueKind.Array:
                builder.Append('[');
                var firstItem = true;
                foreach (var item in element.EnumerateArray())
                {
                    if (!firstItem) builder.Append(',');
                    firstItem = false;
                    Write(item, builder);
                }
                builder.Append(']');
                break;

            /* Numbers go through their raw text: 1 and 1.0 are different bytes and the same
               port, but `GetRawText` keeps whichever the model wrote, and normalising them
               to a double would make 1 and 1.0000000000000002 the same call. Ports are
               integers; this is the conservative half of a rule biased towards refusing. */
            default:
                builder.Append(element.GetRawText());
                break;
        }
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
