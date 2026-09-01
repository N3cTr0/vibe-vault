---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\LocalBrainProbe.cs
---

# tools\EarsTest\LocalBrainProbe.cs

```csharp
using Octavia.Brain;
using Octavia.Core;

/// Not a unit test: talks to whatever local server is configured, if one is up.
/// With no server running it proves the failure message is actionable instead of raw.
internal static class LocalBrainProbe
{
    public static async Task<int> RunAsync()
    {
        var config = OctaviaConfig.Load();
        config.Brain = "local";
        Console.WriteLine($"  endpoint {config.LocalEndpoint}, model {config.LocalModel}");

        using var brain = new LocalBrain(config);
        try
        {
            var said = new List<string>();
            await foreach (var sentence in brain.RespondAsync(new Conversation(), "Say hello in one short sentence."))
            {
                Console.WriteLine($"  said: {sentence}");
                said.Add(sentence);
            }

            if (said.Count == 0) { Console.WriteLine("  FAIL: server answered with nothing"); return 1; }
            Console.WriteLine("  ok   local brain replied");
            return 0;
        }
        catch (InvalidOperationException ex)
        {
            // The expected path on a machine with no server: a message a human can act on.
            Console.WriteLine($"  no server: {ex.Message}");
            var actionable = ex.Message.Contains("Ollama") || ex.Message.Contains("pulled");
            Console.WriteLine(actionable
                ? "  ok   failure message is actionable"
                : "  FAIL: failure message is not actionable");
            return actionable ? 0 : 1;
        }
    }
}
```
