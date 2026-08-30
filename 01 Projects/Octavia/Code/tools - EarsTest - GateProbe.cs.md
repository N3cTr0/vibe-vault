---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\GateProbe.cs
---

# tools\EarsTest\GateProbe.cs

```csharp
using System.Diagnostics;
using Octavia.Brain;
using Octavia.Core;

/// Diagnostic: how well does the gate actually judge, and what does judging cost?
///
/// Whether a sentence was "addressed to an assistant" is a matter of taste, and taste
/// cannot be asserted — so this scores a labelled corpus and prints the disagreements
/// for a person to read. It is the same reasoning as `-- mouth`: the harness proves the
/// properties, the probe shows you the behaviour.
///
/// The two errors are not equal. A **false no** is Octavia ignoring you, which is the
/// one that makes her feel broken. A **false yes** costs a model call. Read the table
/// with that asymmetry in mind before tightening anything.
internal static class GateProbe
{
    /// True means she should answer. Drawn from what a room microphone actually picks
    /// up: television, half a phone call, people talking to each other, and someone
    /// talking to her without saying her name.
    private static readonly (string Text, bool Expected)[] Corpus =
    {
        // Addressed to her, without her name.
        ("what is the weather doing tomorrow", true),
        ("can you turn the volume down a bit", true),
        ("how long does it take to boil an egg", true),
        ("remind me to call the dentist on friday", true),
        ("what was that album we were talking about", true),
        ("is it going to rain before six", true),
        ("tell me a short joke", true),
        ("how do i get rid of a wasp nest", true),

        // Not addressed to her.
        ("coming up after the break, the weather and sport", false),
        ("no i told him that already, he never listens", false),
        ("yeah mate see you sunday, cheers, bye", false),
        ("previously on the show, sarah discovered the letter", false),
        ("and that's a beautiful strike from thirty yards out", false),
        ("i said i'd be there by eight but the traffic was awful", false),
        ("where did i put those keys", false),
        ("right, that's the bins done", false),
        ("so then she said she wasn't coming after all", false),
        ("this episode is brought to you by our sponsors", false),
    };

    public static async Task RunAsync()
    {
        var config = OctaviaConfig.Load();
        Console.WriteLine($"gate model : {config.GateModel}");
        Console.WriteLine($"endpoint   : {config.LocalEndpoint}");
        Console.WriteLine($"names      : {config.WakeNames}");
        Console.WriteLine();

        using var gate = new AttentionGate(new OctaviaConfig
        {
            Gate = "local",
            GateModel = config.GateModel,
            LocalEndpoint = config.LocalEndpoint,
            WakeNames = config.WakeNames,
            // Zero, so the corpus is judged sentence by sentence rather than every line
            // after the first being waved through as a follow-up.
            GateFollowUpSeconds = 0
        });

        int right = 0, falseNo = 0, falseYes = 0;
        var total = Stopwatch.StartNew();
        var costs = new List<double>();

        foreach (var (text, expected) in Corpus)
        {
            var verdict = await gate.JudgeAsync(text);
            costs.Add(verdict.Cost.TotalMilliseconds);

            var agreed = verdict.Answer == expected;
            if (agreed) right++;
            else if (expected) falseNo++;
            else falseYes++;

            var mark = agreed ? "    " : expected ? "MISS" : "COST";
            Console.WriteLine($"  {mark} {(verdict.Answer ? "answer " : "ignore ")} " +
                              $"{verdict.Cost.TotalMilliseconds,5:0} ms  {text}");
            if (!agreed) Console.WriteLine($"         -> gate said: {verdict.Why}");
        }

        total.Stop();
        costs.Sort();

        Console.WriteLine();
        Console.WriteLine($"agreed          {right}/{Corpus.Length}");
        Console.WriteLine($"ignored you     {falseNo}   (the expensive kind — she feels broken)");
        Console.WriteLine($"answered noise  {falseYes}   (the cheap kind — one wasted call)");
        Console.WriteLine();
        Console.WriteLine($"median          {costs[costs.Count / 2]:0} ms");
        Console.WriteLine($"slowest         {costs[^1]:0} ms");
        Console.WriteLine($"whole corpus    {total.Elapsed.TotalSeconds:0.0} s");
        Console.WriteLine();
        Console.WriteLine("Read the latency against what actually reaches the model: anything with her");
        Console.WriteLine("name in it, and anything said within the follow-up window, is settled by the");
        Console.WriteLine("free layer and never gets here. So most of this cost falls on lines she is");
        Console.WriteLine("about to ignore, where nobody is waiting. The number that hurts is the delay");
        Console.WriteLine("before a genuine cold question — the median, not the worst case.");
        Console.WriteLine();
        Console.WriteLine("Beware of tuning the instruction against this corpus. Eighteen lines is far");
        Console.WriteLine("too few to fit to, and a prompt that scores well here and nowhere else is a");
        Console.WriteLine("worse gate than one that reasons correctly and gets two of them wrong.");
    }
}
```
