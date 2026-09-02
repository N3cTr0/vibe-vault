---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\UnifiChecks.cs
---

# tools\EarsTest\UnifiChecks.cs

```csharp
using System.Text.Json;
using Octavia.Brain.Tools;
using Octavia.Core;

/// The UniFi tool server, driven against the real gateway.
///
/// **Skips rather than fails when there is no gateway to talk to**, which is the same
/// bargain the model and microphone probes make: a check that goes red on a machine with no
/// house attached teaches everyone to ignore red.
///
/// The key is read from her own config rather than from a constant here, so the repo never
/// holds a credential and this check tests the entry somebody actually configured.
internal static class UnifiChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;
        void Check(string label, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        var repo = ToolChecks.FindRepoRoot();
        if (repo is null) { Console.WriteLine("  skipped: repo root not found"); return 0; }

        var script = Path.Combine(repo, "tools", "unifi-mcp.ps1");
        if (!File.Exists(script)) { Console.WriteLine($"  skipped: {script} missing"); return 0; }

        var pwsh = ToolChecks.FindPwsh();
        if (pwsh is null) { Console.WriteLine("  skipped: pwsh not found"); return 0; }

        if (Configured() is not { } env)
        {
            Console.WriteLine("  skipped: no 'unifi' server with a key in her config");
            return 0;
        }

        var config = new OctaviaConfig
        {
            McpServers = new Dictionary<string, McpServer>
            {
                ["unifi"] = new()
                {
                    Command = pwsh,
                    Args = ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", script],
                    Env = env
                }
            }
        };

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        Check("the server started", registry.Any);
        if (!registry.Any) return failures;

        var tools = await registry.ListAsync();
        Check("all five tools were listed", tools.Count == 5, $"{tools.Count}");
        Check("names are namespaced", tools.All(t => t.Name.StartsWith("unifi__")),
              string.Join(", ", tools.Select(t => t.Name)));

        /* The check this file exists for.

           Every tool here reads and changes nothing, so every one must classify as Read —
           and that is not a property of the code, it is a property of the *wording*.
           `RiskOf` looks for its dangerous words first, so a description that gains a
           "restart", a "reset" or an "order" would quietly turn a network status query into
           something she stops to ask permission for. Nothing else would notice. */
        var wrong = tools.Where(t => t.Risk != ToolRisk.Read).ToList();
        Check("every tool is judged a read", wrong.Count == 0,
              string.Join(", ", wrong.Select(t => $"{t.Name} is {t.Risk}")));

        var status = await registry.CallAsync("unifi__get_status", Empty());
        Check("the gateway answers", status.Contains("client(s) connected"), Head(status));

        var devices = await registry.CallAsync("unifi__list_devices", Empty());
        Check("the hardware is named", devices.Contains("firmware"), Head(devices));

        var cameras = await registry.CallAsync("unifi__list_cameras", Empty());
        Check("Protect answers on the same key", cameras.Contains("camera"), Head(cameras));

        // A search that cannot match, so this asserts the empty answer rather than whatever
        // happens to be plugged in on the day.
        var none = await registry.CallAsync(
            "unifi__find_client", Args("""{"query":"zzz-nothing-is-called-this"}"""));

        Check("a search with no hits says so", none.Contains("Nothing connected matches"), Head(none));

        return failures;
    }

    /// Her configured `unifi` server, or null when there is not one to test.
    private static Dictionary<string, string>? Configured()
    {
        try
        {
            var file = Path.Combine(Paths.DataDir, "config.json");
            if (!File.Exists(file)) return null;

            using var json = JsonDocument.Parse(File.ReadAllText(file));

            if (!json.RootElement.TryGetProperty("McpServers", out var servers) ||
                !servers.TryGetProperty("unifi", out var unifi) ||
                !unifi.TryGetProperty("Env", out var env))
                return null;

            var read = env.EnumerateObject()
                          .ToDictionary(p => p.Name, p => p.Value.GetString() ?? "");

            return read.TryGetValue("UNIFI_API_KEY", out var key) && key.Length > 0 ? read : null;
        }
        catch
        {
            return null;
        }
    }

    private static JsonElement Empty() => Args("{}");

    private static JsonElement Args(string json) => JsonDocument.Parse(json).RootElement.Clone();

    /// One line, so a failure is readable without a wall of network inventory behind it.
    private static string Head(string text) =>
        text.Split('\n')[0] is { Length: > 90 } long_ ? long_[..90] + "..." : text.Split('\n')[0];
}
```
