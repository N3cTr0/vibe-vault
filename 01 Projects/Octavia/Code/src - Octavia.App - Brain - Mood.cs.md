---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Brain\Mood.cs
---

# src\Octavia.App\Brain\Mood.cs

```csharp
using System.Text.RegularExpressions;

namespace Octavia.Brain;

/// <param name="Name">A VRM 1.0 expression preset, so the face needs no translation.</param>
internal readonly record struct Mood(string Name, double Weight)
{
    public static readonly Mood Neutral = new("neutral", 0);
}

/// What her face does while she says a sentence.
///
/// Read from the text here rather than asked of the model: it is free, it is instant,
/// and it obeys the standing rule that anything reflex-speed stays local. The `emotion`
/// message exists so a model can override this later; until then a sentence that says
/// "I'm sorry" should not be delivered with a pleasant blank stare.
internal static partial class Moods
{
    private static readonly (string[] Words, string Mood, double Weight)[] Cues =
    [
        (["sorry", "unfortunately", "afraid not", "i can't", "i cannot", "sadly", "regret", "went wrong", "failed"],
            "sad", 0.7),
        (["glad", "lovely", "wonderful", "delighted", "happy to", "of course", "you're welcome", "nicely done", "excellent"],
            "happy", 0.7),
        (["oh!", "wow", "really?", "goodness", "that's surprising", "had no idea"],
            "surprised", 0.8)
    ];

    public static Mood Read(string sentence)
    {
        if (string.IsNullOrWhiteSpace(sentence)) return Mood.Neutral;

        var text = sentence.ToLowerInvariant();

        foreach (var (words, mood, weight) in Cues)
        {
            if (words.Any(word => text.Contains(word, StringComparison.Ordinal)))
                return new Mood(mood, weight);
        }

        // Punctuation is a weaker signal than a word, so it only ever colours her
        // slightly — a face that beams at every exclamation mark reads as unhinged.
        if (text.EndsWith('!')) return new Mood("happy", 0.4);
        if (Question().IsMatch(text)) return new Mood("relaxed", 0.35);

        return Mood.Neutral;
    }

    [GeneratedRegex(@"\?\s*$")]
    private static partial Regex Question();
}
```
