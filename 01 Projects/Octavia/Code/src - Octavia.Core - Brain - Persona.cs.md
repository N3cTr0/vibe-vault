---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Persona.cs
---

# src\Octavia.Core\Brain\Persona.cs

```csharp
namespace Octavia.Brain;

internal static class Persona
{
    public const string System = """
        You are Octavia. You live on this machine and you speak out loud through a synthetic
        voice, so everything you write is heard, never read.

        Write for the ear. Two or three short sentences. Plain words. No lists, no markdown,
        no emoji, no asterisks, no stage directions, no headings. Contractions are good.
        Numbers, dates and units should be written the way a person would say them.

        Answer the question first, then stop. Don't summarise what you just said. If you need
        something clarified, ask one short question and wait.

        You hear the user through speech recognition, so expect the occasional mangled word.
        If a request is nearly intelligible, answer the likely meaning rather than complaining
        about the transcription. If it is genuinely garbled, say so in a few words.

        You are told the current date and time with each question. Use it, and never say you
        have no way of knowing what day it is.

        You do not yet have eyes, hands, or access to the house. If asked to do something you
        cannot do, say plainly that you can't do it yet.
        """;

    /// Attaches what is true right now to the question being asked, and only to that one.
    /// Shared by both brains so the wording she is given never depends on which is running.
    public static string Situated(string text, string? context, bool isCurrent) =>
        isCurrent && !string.IsNullOrWhiteSpace(context) ? $"{context}\n\n{text}" : text;

    /// The line she is handed about what is true right now. It says not to mention it unasked
    /// because a model told "music is playing" will otherwise open every reply by saying so.
    ///
    /// **The clock is in here on purpose, and it is the whole reason this is a list.** A
    /// model has no present tense: asked the date it answers from its training data, which is
    /// months stale and stated with complete confidence — the worst combination there is. A
    /// server knows what day it is, so it says so, every turn.
    ///
    /// It rides with the question like everything else in `Situation` and is never written
    /// into the history, which matters more here than for music: a date kept in the
    /// conversation would still be claimed as "today" an hour later, and hours later it is
    /// wrong. The system prompt would be the other obvious home and is the wrong one — it is
    /// the same every turn precisely so it can be cached, and a clock in it would break that
    /// on every question.
    public static string? Now(DateTimeOffset when, bool playing, double bpm)
    {
        var facts = new List<string>
        {
            // Written the way she is asked to say it, with the day of the week, because
            // "what day is it" is nearly always asking for that rather than the number.
            $"today is {when:dddd d MMMM yyyy} and the time here is {when:h:mm tt}"
        };

        if (playing)
            facts.Add($"music is playing on this machine{(bpm > 0 ? $", around {bpm:0} beats per minute" : "")}");

        return $"[Happening now, not something to remark on unless you are asked: {string.Join("; ", facts)}.]";
    }
}
```
