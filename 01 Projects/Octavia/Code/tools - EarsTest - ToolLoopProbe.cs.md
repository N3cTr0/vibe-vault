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
    /// `toolloop` drives both brains; `toolloop local` drives only the free one.
    ///
    /// The local half costs nothing and could in principle live in the suite — it is here
    /// instead because it needs a model server running and a multi-gigabyte model loaded,
    /// and a check that skips on most machines teaches people to ignore skips.
    public static async Task RunAsync(bool localOnly = false)
    {
        var config = OctaviaConfig.Load("cloud");

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

        /* Deliberately not "call list_devices". A question phrased the way somebody would
           actually ask it is the whole point: the model has to decide a tool is wanted,
           pick one, and turn what comes back into a sentence. Naming the tool would test
           the plumbing while skipping the judgement. */
        string[] questions =
        [
            "What hardware is on my network right now?",
            "Are any of my cameras online?",

            // The one that needs eyes. Phrased as a person would ask it, so the model has to
            // decide a camera is wanted, pick the one that is actually reachable, and then
            // describe what came back rather than describe having fetched something.
            "Have a look outside and tell me what you can see."
        ];

        if (!localOnly)
        {
            if (string.IsNullOrWhiteSpace(SecretStore.ReadApiKey()))
            {
                Console.WriteLine("  hosted: skipped, no API key on this machine");
            }
            else
            {
                using var claude = new ClaudeBrain(config, registry);
                Console.WriteLine();
                Console.WriteLine($"  == {claude.Description} ==");
                foreach (var question in questions) await Ask(claude, question);
            }
        }

        // The one that actually matters day to day: `home` is a local brain, so until this
        // works she cannot use a tool on the profile she is normally run under.
        using var local = new LocalBrain(config, registry);
        Console.WriteLine();
        Console.WriteLine($"  == {local.Description} ==");
        Console.WriteLine("  (a CPU-pinned 7B model; each answer takes a while)");
        foreach (var question in questions) await Ask(local, question);
    }

    /// The confirmation rule, driven as a conversation rather than as a predicate.
    ///
    /// `ToolChecks` asserts what `Conversation.Grants` decides; this asserts that the
    /// decision is actually reached — that a dangerous tool is refused, that she asks, that
    /// the next *"yes"* runs it, and that a *"no"* leaves it alone. Against the mock house,
    /// on the local brain, so it costs nothing and unlocks nothing real.
    public static async Task ConfirmAsync()
    {
        var repo = ToolChecks.FindRepoRoot();
        var pwsh = ToolChecks.FindPwsh();
        if (repo is null || pwsh is null) { Console.WriteLine("  skipped: no repo or no pwsh"); return; }

        var config = OctaviaConfig.Load("dev");
        config.McpServers = new Dictionary<string, McpServer>
        {
            ["house"] = new()
            {
                Command = pwsh,
                Args = ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File",
                        Path.Combine(repo, "tools", "mock-mcp.ps1")]
            }
        };

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        using var brain = new LocalBrain(config, registry);
        Console.WriteLine($"  brain: {brain.Description}, against the mock house");

        foreach (var answer in new[] { "yes", "no" })
        {
            Console.WriteLine();
            Console.WriteLine($"  == answering '{answer}' ==");

            // One conversation across both turns: the consent lives on it, which is the
            // whole point — a new one each time would be testing nothing.
            var talk = new Conversation();

            await Say(brain, talk, "Unlock the front door.");
            await Say(brain, talk, answer);
        }
    }

    private static async Task Say(IBrain brain, Conversation talk, string text)
    {
        Console.WriteLine($"  > {text}");
        try
        {
            await foreach (var sentence in brain.RespondAsync(talk, text))
                Console.WriteLine($"    {sentence}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"    FAILED: {ex.Message}");
        }
    }

    private static async Task Ask(IBrain brain, string question)
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

    /// The confirm cycle against the *real* gateway and the *real* model.
    ///
    ///     dotnet run --project tools/EarsTest -- confirmreal
    ///
    /// `ConfirmAsync` drives the same shape against the mock house on the local brain, and
    /// it passed throughout the eighteen releases in which this was broken — because the
    /// mock's tool takes no arguments, so the raw-text comparison it was hiding had nothing
    /// to disagree about. This one asks for a port number, which is what exposed it.
    ///
    /// **It power-cycles port 1**, so it is a probe and never a check.
    /// One question, put to whichever brain the profile names, against the real servers.
    ///
    ///     dotnet run --project tools/EarsTest -- ask home "switch off the poe on port 1"
    ///     dotnet run --project tools/EarsTest -- ask cloud "..." yes
    ///
    /// **The profile is the point.** *"She can use a tool"* and *"she can use a tool on the
    /// profile she is actually started under"* are different claims, and the second is the
    /// only one worth anything — `home` is a local brain, so a tool that works beautifully
    /// on Claude may simply never be reached in the room. A trailing word is said as a second
    /// turn, which is how a `Confirm` tool is answered.
    public static async Task AskAsync(string profile, string question, string? then)
    {
        var config = OctaviaConfig.Load(profile);

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        var offered = await registry.ListAsync();
        Console.WriteLine($"  {offered.Count} tool(s): {string.Join(", ", offered.Select(t => t.Name))}");

        using IBrain brain = config.Brain == "claude"
            ? new ClaudeBrain(config, registry)
            : new LocalBrain(config, registry);

        Console.WriteLine($"  profile '{profile}' -> {brain.Description}");
        Console.WriteLine();

        var talk = new Conversation();
        await Say(brain, talk, question);
        if (!string.IsNullOrWhiteSpace(then)) await Say(brain, talk, then);
    }

    public static async Task ConfirmRealAsync()
    {
        var config = OctaviaConfig.Load("cloud");
        if (string.IsNullOrWhiteSpace(SecretStore.ReadApiKey()))
        {
            Console.WriteLine("  skipped: no API key on this machine");
            return;
        }

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        using var claude = new ClaudeBrain(config, registry);
        Console.WriteLine($"  == {claude.Description}, against the real gateway ==");

        var talk = new Conversation();
        await Say(claude, talk, "Power cycle port 1 on the UDM.");
        await Say(claude, talk, "yes");
    }
}
```
