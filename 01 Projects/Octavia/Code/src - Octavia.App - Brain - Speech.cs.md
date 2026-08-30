---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Brain\Speech.cs
---

# src\Octavia.App\Brain\Speech.cs

```csharp
using System.Text;
using System.Text.RegularExpressions;

namespace Octavia.Brain;

/// Turning a token stream into things worth saying out loud.
internal static partial class Speech
{
    /// Pulls every complete sentence out of the buffer, leaving the partial tail.
    public static IEnumerable<string> DrainSentences(StringBuilder pending)
    {
        while (true)
        {
            var buffer = pending.ToString();
            var cut = -1;

            for (var i = 0; i < buffer.Length; i++)
            {
                if (buffer[i] is not ('.' or '!' or '?' or '\n')) continue;
                // A period between digits is a decimal point, not the end of a thought.
                if (buffer[i] == '.' && i > 0 && i + 1 < buffer.Length &&
                    char.IsDigit(buffer[i - 1]) && char.IsDigit(buffer[i + 1])) continue;
                cut = i;
                break;
            }

            if (cut < 0) yield break;

            var sentence = buffer[..(cut + 1)].Trim();
            pending.Remove(0, cut + 1);
            if (sentence.Length > 0) yield return sentence;
        }
    }

    [GeneratedRegex(@"```[\s\S]*?```|`([^`]*)`")] private static partial Regex Code();
    [GeneratedRegex(@"^\s{0,3}#{1,6}\s*", RegexOptions.Multiline)] private static partial Regex Heading();
    [GeneratedRegex(@"^\s*[-*+]\s+", RegexOptions.Multiline)] private static partial Regex Bullet();
    [GeneratedRegex(@"\*{1,3}([^*]+)\*{1,3}|_{1,3}([^_]+)_{1,3}")] private static partial Regex Emphasis();
    [GeneratedRegex(@"\[([^\]]*)\]\([^)]*\)")] private static partial Regex Link();
    [GeneratedRegex(@"[ \t]{2,}")] private static partial Regex Runs();

    /// Small models ignore "no markdown, no stage directions" no matter how firmly
    /// they are asked. Rather than let asterisks and hashes be read aloud, strip them.
    public static string Speakable(string text)
    {
        if (string.IsNullOrWhiteSpace(text)) return string.Empty;

        text = Code().Replace(text, "$1");
        text = Heading().Replace(text, string.Empty);
        text = Bullet().Replace(text, string.Empty);
        text = Link().Replace(text, "$1");
        text = Emphasis().Replace(text, "$1$2");
        text = Runs().Replace(text, " ");

        return text.Trim();
    }

    /// The same job as ThinkFilter, for a reply that arrived whole rather than in
    /// chunks — a one-shot request has nothing to stream, and a state machine that
    /// holds back a partial tag would hold back the end of the answer.
    ///
    /// An unclosed &lt;think&gt; discards the remainder, because a model still thinking
    /// when it ran out of tokens never reached its answer.
    public static string WithoutThinking(string text)
    {
        var output = new StringBuilder();
        var at = 0;

        while (at < text.Length)
        {
            var start = text.IndexOf("<think>", at, StringComparison.OrdinalIgnoreCase);
            if (start < 0) { output.Append(text[at..]); break; }

            output.Append(text[at..start]);
            var end = text.IndexOf("</think>", start, StringComparison.OrdinalIgnoreCase);
            if (end < 0) break;
            at = end + "</think>".Length;
        }

        return output.ToString().Trim();
    }

    /// Reasoning models emit their scratchpad inline. Discards anything between
    /// <think> and </think>, across chunk boundaries, and buffers a partial tag
    /// rather than speaking it.
    public sealed class ThinkFilter
    {
        private const string Open = "<think>";
        private const string Close = "</think>";

        private readonly StringBuilder _held = new();
        private bool _thinking;

        public string Filter(string chunk)
        {
            _held.Append(chunk);
            var output = new StringBuilder();

            while (_held.Length > 0)
            {
                var buffer = _held.ToString();

                if (_thinking)
                {
                    var end = buffer.IndexOf(Close, StringComparison.OrdinalIgnoreCase);
                    if (end < 0)
                    {
                        Keep(buffer, Close.Length - 1);
                        break;
                    }

                    _thinking = false;
                    _held.Remove(0, end + Close.Length);
                    continue;
                }

                var start = buffer.IndexOf(Open, StringComparison.OrdinalIgnoreCase);
                if (start >= 0)
                {
                    output.Append(buffer[..start]);
                    _thinking = true;
                    _held.Remove(0, start + Open.Length);
                    continue;
                }

                // Hold back only a trailing run that could still grow into "<think>".
                // Holding a fixed margin instead would stall short replies entirely.
                var hold = 0;
                for (var n = Math.Min(Open.Length - 1, buffer.Length); n > 0; n--)
                {
                    if (string.Compare(buffer, buffer.Length - n, Open, 0, n,
                            StringComparison.OrdinalIgnoreCase) == 0)
                    {
                        hold = n;
                        break;
                    }
                }

                var safe = buffer.Length - hold;
                if (safe <= 0) break;

                output.Append(buffer[..safe]);
                _held.Remove(0, safe);
                break;
            }

            return output.ToString();
        }

        /// Whatever is still held once the stream ends, minus any unclosed think block.
        public string Flush()
        {
            var rest = _thinking ? string.Empty : _held.ToString();
            _held.Clear();
            _thinking = false;
            return rest;
        }

        private void Keep(string buffer, int tail)
        {
            if (buffer.Length <= tail) return;
            _held.Remove(0, buffer.Length - tail);
        }
    }
}
```
