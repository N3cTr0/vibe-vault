---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\ToolLoopProbe.cs
---

# tools\EarsTest\ToolLoopProbe.cs

```csharp
using Octavia.Brain;
using Octavia.Brain.Tools;
using Octavia.Core;

/// The brain-side tool loop, driven end to end against the real API and the real gateway.
///
/// **A probe rather than a check, and never part of the suite.** Every run of this spends
/// money on a paid model, and *"a self-test that spends money is a bad self-test"* is
/// already written down here — anything a worried person might press repeatedly has to be
/// free. It is invoked deliberately:
///
///     dotnet run --project tools/EarsTest -- toolloop
///
/// What it proves is the one thing nothing else can: that a question in English reaches a
/// tool, the answer comes back, and she says something that could only have come from the
/// gateway. Everything up to the last hop is covered by `ToolChecks` and `UnifiChecks` for
/// nothing.
internal static class ToolLoopProbe
{
    public static async Task RunAsync()
    {
        var config = OctaviaConfig.Load("cloud");

        if (string.IsNullOrWhiteSpace(SecretStore.ReadApiKey()))
        {
            Console.WriteLine("  skipped: no API key on this machine");
            return;
        }

        if (config.McpServers.Count == 0)
        {
            Console.WriteLine("  skipped: no tool servers configured");
            return;
        }

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        var offered = await registry.ListAsync();
        Console.WriteLine($"  {offered.Count} tool(s) offered: {string.Join(", ", offered.Select(t => t.Name))}");

        if (offered.Count == 0)
        {
            Console.WriteLine("  nothing to call; stopping before spending anything");
            return;
        }

        using var brain = new ClaudeBrain(config, registry);
        Console.WriteLine($"  brain: {brain.Description}");

        /* Deliberately not "call list_devices". A question phrased the way somebody would
           actually ask it is the whole point: the model has to decide a tool is wanted,
           pick one, and turn what comes back into a sentence. Naming the tool would test
           the plumbing while skipping the judgement. */
        await Ask(brain, "What hardware is on my network right now?");
        await Ask(brain, "Are any of my cameras online?");
    }

    private static async Task Ask(ClaudeBrain brain, string question)
    {
        Console.WriteLine();
        Console.WriteLine($"  > {question}");

        var history = new Conversation();
        var said = new List<string>();

        try
        {
            await foreach (var sentence in brain.RespondAsync(history, question))
            {
                said.Add(sentence);
                Console.WriteLine($"    {sentence}");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"    FAILED: {ex.Message}");
            return;
        }

        if (said.Count == 0) Console.WriteLine("    FAILED: she said nothing at all");
    }
}
```
