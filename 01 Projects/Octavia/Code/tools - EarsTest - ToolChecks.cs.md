---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\ToolChecks.cs
---

# tools\EarsTest\ToolChecks.cs

```csharp
using System.Text.Json;
using Octavia.Brain.Tools;
using Octavia.Brain;
using Octavia.Core;

/// The tool seam, driven against the mock MCP server in tools\mock-mcp.ps1.
///
/// This exercises the real client — a real child process, real JSON-RPC over stdio, real
/// framing — rather than a stub of it, because every part of MCP that can go wrong is in
/// the transport rather than in the shape of the data.
internal static class ToolChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;
        void Check(string label, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        var repo = FindRepoRoot();
        if (repo is null) { Console.WriteLine("  skipped: repo root not found"); return 0; }

        var script = Path.Combine(repo, "tools", "mock-mcp.ps1");
        if (!File.Exists(script)) { Console.WriteLine($"  skipped: {script} missing"); return 0; }

        var pwsh = FindPwsh();
        if (pwsh is null) { Console.WriteLine("  skipped: pwsh not found"); return 0; }

        var config = new OctaviaConfig
        {
            McpServers = new Dictionary<string, McpServer>
            {
                ["house"] = new()
                {
                    Command = pwsh,
                    Args = ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", script]
                }
            }
        };

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        Check("the server started", registry.Any);
        if (!registry.Any) return failures;

        var tools = await registry.ListAsync();
        Check("tools were listed", tools.Count == 3, $"{tools.Count}");

        // Names are prefixed by server, so two integrations may both offer get_state.
        Check("names are namespaced", tools.All(t => t.Name.StartsWith("house__")),
            string.Join(", ", tools.Select(t => t.Name)));

        Check("schemas survived", tools.All(t => t.Schema.ContainsKey("properties")));

        var read = tools.FirstOrDefault(t => t.Name.EndsWith("get_state"));
        var act = tools.FirstOrDefault(t => t.Name.EndsWith("set_light"));
        var danger = tools.FirstOrDefault(t => t.Name.EndsWith("unlock_door"));

        Check("a read is judged a read", read?.Risk == ToolRisk.Read, $"{read?.Risk}");
        Check("a light is judged an act", act?.Risk == ToolRisk.Act, $"{act?.Risk}");
        Check("an unlock demands confirming", danger?.Risk == ToolRisk.Confirm, $"{danger?.Risk}");

        var stateArgs = JsonDocument.Parse("""{"entity":"the hall lamp"}""").RootElement;
        var answer = (await registry.CallAsync("house__house_get_state", stateArgs)).Text;
        Check("a read runs and answers", answer.Contains("the hall lamp"), answer);

        // The rule that matters most: dangerous tools do not run on their own.
        var unlock = (await registry.CallAsync("house__house_unlock_door", stateArgs)).Text;
        Check("an unlock is refused unconfirmed", !unlock.Contains("Unlocked"), unlock);

        var confirmed = (await registry.CallAsync("house__house_unlock_door", stateArgs, confirmed: true)).Text;
        Check("an unlock runs once confirmed", confirmed.Contains("Unlocked"), confirmed);

        // A name nobody offers must be an answer, not an exception: the model has to be
        // able to be told it was wrong.
        var missing = (await registry.CallAsync("house__nonsense", stateArgs)).Text;
        Check("an unknown tool answers rather than throws", missing.Contains("no tool"), missing);

        /* A spoken yes, and how narrowly it counts.

           This is the rule that decides whether a sentence can open a door, so it is checked
           in every direction that could go wrong rather than only the happy one. The bias
           throughout is towards refusing: a yes misread as a no costs one repeated question,
           and a no misread as a yes costs whatever the tool does. */
        const string door = "house__house_unlock_door";
        const string back = """{"entity":"the back door"}""";
        const string front = """{"entity":"the front door"}""";

        Conversation Asked()
        {
            var talk = new Conversation();
            talk.AwaitYes(door, back);
            return talk;
        }

        Check("a plain yes counts",
              Conversation.Grants(Asked().TakeConsent(), door, back, "yes"));

        Check("so does go ahead",
              Conversation.Grants(Asked().TakeConsent(), door, back, "go ahead"));

        Check("and a yes with words around it",
              Conversation.Grants(Asked().TakeConsent(), door, back, "Yes, please do that."));

        Check("a no does not",
              !Conversation.Grants(Asked().TakeConsent(), door, back, "no"));

        // The one a looser rule gets wrong: agreement and refusal in the same breath.
        Check("a yes with a caveat does not",
              !Conversation.Grants(Asked().TakeConsent(), door, back, "yes, but not yet"));

        Check("silence on the subject does not",
              !Conversation.Grants(Asked().TakeConsent(), door, back, "what is the weather like"));

        // Consent is to a *call*, not to a tool. Saying yes about the back door must not
        // unlock the front one, however the model phrases its second attempt.
        Check("a yes about one thing is not a yes about another",
              !Conversation.Grants(Asked().TakeConsent(), door, front, "yes"));

        Check("a yes to one tool is not a yes to another",
              !Conversation.Grants(Asked().TakeConsent(), "house__house_set_light", back, "yes"));

        // The lifetime: taking it clears it, so a second turn cannot reuse the first's yes.
        var once = Asked();
        var first = once.TakeConsent();
        var second = once.TakeConsent();
        Check("consent survives exactly one turn",
              Conversation.Grants(first, door, back, "yes") &&
              !Conversation.Grants(second, door, back, "yes"));

        Check("and nothing is pending before she asks",
              !Conversation.Grants(new Conversation().TakeConsent(), door, back, "yes"));

        return failures;
    }

    internal static string? FindRepoRoot()
    {
        var dir = new DirectoryInfo(AppContext.BaseDirectory);
        while (dir is not null)
        {
            if (File.Exists(Path.Combine(dir.FullName, "Octavia.slnx"))) return dir.FullName;
            dir = dir.Parent;
        }
        return null;
    }

    internal static string? FindPwsh()
    {
        var candidates = new[]
        {
            @"C:\Program Files\PowerShell\7\pwsh.exe",
            Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                         @"Programs\PowerShell7\pwsh.exe")
        };

        return candidates.FirstOrDefault(File.Exists);
    }
}
```
