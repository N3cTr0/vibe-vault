---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\BrainChecks.cs
---

# tools\EarsTest\BrainChecks.cs

```csharp
using System.Text;
using Octavia.Brain;

internal static class BrainChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, string actual, string expected)
        {
            var ok = actual == expected;
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}: '{actual}'");
            if (!ok)
            {
                Console.WriteLine($"       expected: '{expected}'");
                failures++;
            }
        }

        Console.WriteLine("think filter (streamed in chunks):");

        // A reasoning model's scratchpad must never reach the voice, even when the
        // tags are split across chunk boundaries.
        Check("whole tag in one chunk",
            Feed(["<think>hmm let me see</think>Hello there."]),
            "Hello there.");

        Check("tag split mid-token",
            Feed(["<thi", "nk>secret", " reasoning</thi", "nk>Out loud."]),
            "Out loud.");

        Check("text before and after",
            Feed(["Sure. <think>why</think>Here it is."]),
            "Sure. Here it is.");

        Check("no think tags at all",
            Feed(["Just ", "a plain ", "reply."]),
            "Just a plain reply.");

        Check("unclosed think block is dropped",
            Feed(["Fine.<think>never closed"]),
            "Fine.");

        Check("lone angle bracket survives",
            Feed(["5 < 6 is true."]),
            "5 < 6 is true.");

        // Regression: a reply shorter than the opening tag must not be swallowed.
        Check("very short reply is not held",
            Feed(["Hi."]),
            "Hi.");

        Check("text emitted without waiting for more",
            new Speech.ThinkFilter().Filter("Hello there."),
            "Hello there.");

        Check("only a real tag prefix is held back",
            new Speech.ThinkFilter().Filter("All done.<"),
            "All done.");

        Console.WriteLine("markdown stripping:");

        Check("bold and italics",
            Speech.Speakable("That is **very** _important_ indeed."),
            "That is very important indeed.");

        Check("heading and bullet",
            Speech.Speakable("## Notes\n- first point"),
            "Notes\nfirst point");

        Check("inline code",
            Speech.Speakable("Run `dotnet build` now."),
            "Run dotnet build now.");

        Check("link text kept, url dropped",
            Speech.Speakable("See [the docs](https://example.com) for more."),
            "See the docs for more.");

        Console.WriteLine("sentence splitting:");

        var pending = new StringBuilder("One. Two! Three? And a half");
        var sentences = Speech.DrainSentences(pending).ToList();
        Check("complete sentences", string.Join("|", sentences), "One.|Two!|Three?");
        Check("partial tail held", pending.ToString().Trim(), "And a half");

        var decimals = new StringBuilder("It costs 3.50 today.");
        Check("decimal point is not a sentence end",
            string.Join("|", Speech.DrainSentences(decimals)),
            "It costs 3.50 today.");

        return failures;
    }

    private static string Feed(string[] chunks)
    {
        var filter = new Speech.ThinkFilter();
        var output = new StringBuilder();
        foreach (var chunk in chunks) output.Append(filter.Filter(chunk));
        output.Append(filter.Flush());
        return output.ToString();
    }
}
```
