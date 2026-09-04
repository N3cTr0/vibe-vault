---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\SpeechChecks.cs
---

# tools\EarsTest\SpeechChecks.cs

```csharp
using System.Text;
using Octavia.Brain;

/// How her answer is cut into the sentences she speaks.
///
/// **Fed a token at a time, because that is the only way the bug appears.** Handed the whole
/// answer at once, the splitter has always got IP addresses right — the digit after the
/// period is right there. Streaming, it is not: `100.` arrives as a complete buffer and the
/// deciding digit is still in flight. That produced *"the IP address 100. 103. 17. 84."*,
/// which she said as six sentences with a pause between each, and which flickered the caption
/// six times on the way past.
internal static class SpeechChecks
{
    public static int Run()
    {
        var failures = 0;
        void Check(string label, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        var whole = Streamed("The gateway is at 100.103.17.84 and runs firmware 5.1.31.");
        Check("an IP address is not four sentences", whole.Count == 1,
              string.Join(" | ", whole));

        var two = Streamed("It is online. The address is 10.1.30.11.");
        Check("two real sentences are still two", two.Count == 2, string.Join(" | ", two));

        var version = Streamed("It runs firmware 6.9.1. The uptime is fifteen days.");
        Check("a version number ending a sentence still ends it", version.Count == 2,
              string.Join(" | ", version));

        var plain = Streamed("Hello there. How are you? Good!");
        Check("ordinary sentences are untouched", plain.Count == 3, string.Join(" | ", plain));

        var trailing = Streamed("The port number is 8848.");
        Check("an answer ending in a number is not lost", trailing.Count == 1,
              string.Join(" | ", trailing));

        return failures;
    }

    /// One character at a time, which is harsher than any real tokeniser and therefore the
    /// right test: every intermediate buffer a model could produce is exercised.
    private static List<string> Streamed(string answer)
    {
        var said = new List<string>();
        var pending = new StringBuilder();

        foreach (var c in answer)
        {
            pending.Append(c);
            said.AddRange(Speech.DrainSentences(pending));
        }

        // What the brains do when the stream ends — without this the tail is never spoken.
        var tail = pending.ToString().Trim();
        if (tail.Length > 0) said.Add(tail);

        return said;
    }
}
```
