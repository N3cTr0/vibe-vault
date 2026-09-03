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
        Check("all eight tools were listed", tools.Count == 8, $"{tools.Count}");
        Check("names are namespaced", tools.All(t => t.Name.StartsWith("unifi__")),
              string.Join(", ", tools.Select(t => t.Name)));

        /* The check this file exists for, and it cuts both ways now.

           Risk is not a property of the code, it is a property of the *wording*: `RiskOf`
           reads the name and description and looks for its dangerous words first. That used
           to mean one hazard — a description gaining a "restart" or a "reset" would turn a
           status query into something she stops to ask permission for.

           Since `power_cycle_port` the opposite hazard is the serious one. **The only thing
           standing between "restart the power on port 4" and her doing it unasked is the
           word "Restart" in that description.** Reword it to something gentler and the
           classification silently drops to `Act`, which she may perform on her own. So both
           directions are pinned here: the reads must stay reads, and the one write must stay
           a `Confirm`. */
        var shouldRead = tools.Where(t => t.Name != "unifi__power_cycle_port" && t.Risk != ToolRisk.Read).ToList();
        Check("every read is judged a read", shouldRead.Count == 0,
              string.Join(", ", shouldRead.Select(t => $"{t.Name} is {t.Risk}")));

        var cycle = tools.FirstOrDefault(t => t.Name == "unifi__power_cycle_port");
        Check("cutting power to a port is judged a confirm", cycle?.Risk == ToolRisk.Confirm,
              cycle is null ? "the tool is missing" : $"it is {cycle.Risk}");

        var status = (await registry.CallAsync("unifi__get_status", Empty())).Text;
        Check("the gateway answers", status.Contains("client(s) connected"), Head(status));

        var devices = (await registry.CallAsync("unifi__list_devices", Empty())).Text;
        Check("the hardware is named", devices.Contains("firmware"), Head(devices));

        var cameras = (await registry.CallAsync("unifi__list_cameras", Empty())).Text;
        Check("Protect answers on the same key", cameras.Contains("camera"), Head(cameras));

        var ports = (await registry.CallAsync("unifi__list_ports", Empty())).Text;
        Check("the ports are listed with their PoE state", ports.Contains("PoE"), Head(ports));
        Check("...and it says it cannot see what is on them",
              ports.Contains("does not report which client is on which port"), Head(ports));

        /* **Nothing here ever cuts power to a real port**, and that is deliberate rather than
           an omission. A green suite must not be able to reboot a camera, and a test that
           power-cycles whatever happens to be on port 4 today is a test nobody dares run
           twice.

           Two things are worth asserting, and the first one caught the second being written
           wrongly: an earlier version of this called the tool with a bad port and expected
           the script's refusal, and got the *registry's* refusal instead, because
           `power_cycle_port` never reached the script at all without a yes. That is the
           guard working, so it is now the check. */
        var unasked = (await registry.CallAsync(
            "unifi__power_cycle_port", Args("""{"port":4}"""))).Text;

        Check("power is not cut without a yes", unasked.Contains("needs the person to say yes"),
              Head(unasked));

        /* And with the yes given, the script's own refusals - which stop one line before the
           POST, so they exercise argument handling, device resolution and the port lookup
           without touching anything. `confirmed: true` is the flag a real turn sets only
           after somebody has actually said it out loud; see `Conversation.Grants`. */
        var noPort = (await registry.CallAsync(
            "unifi__power_cycle_port", Args("""{"port":99}"""), confirmed: true)).Text;

        Check("...and a port that does not exist is refused", noPort.Contains("no port 99"), Head(noPort));

        var noPoe = (await registry.CallAsync(
            "unifi__power_cycle_port", Args("""{"port":9}"""), confirmed: true)).Text;

        Check("...and so is a port with no PoE on it",
              noPoe.Contains("does not supply PoE"), Head(noPoe));

        /* A device name that matches nothing. This line existed and was broken: it read
           `return if (...) {...} else {...}`, which PowerShell *parses* — `if` is taken as a
           command name — and then fails at runtime with "the term 'if' is not recognized".
           A parse check said the file was clean. Only running it says otherwise. */
        var noDevice = (await registry.CallAsync(
            "unifi__power_cycle_port",
            Args("""{"port":4,"device":"zzz-no-such-switch"}"""), confirmed: true)).Text;

        Check("...and a device name that matches nothing",
              noDevice.Contains("No PoE device matches"), Head(noDevice));

        // A search that cannot match, so this asserts the empty answer rather than whatever
        // happens to be plugged in on the day.
        var none = (await registry.CallAsync(
            "unifi__find_client", Args("""{"query":"zzz-nothing-is-called-this"}"""))).Text;

        Check("a search with no hits says so", none.Contains("Nothing connected matches"), Head(none));

        /* The camera, which is the one tool whose answer is not words.

           A name nobody has and a camera that is present but unreachable are different
           problems with different answers, and both are asserted: the first is a typo and
           the second is a fact about the house, so telling somebody "not reachable" for a
           name that was never right would send them looking in the wrong place. */
        var nobody = (await registry.CallAsync(
            "unifi__look_at_camera", Args("""{"camera":"zzz-no-such-camera"}"""))).Text;

        Check("an unknown camera is named as unknown",
              nobody.Contains("no camera called"), Head(nobody));

        /* Whether a picture comes back depends on what is plugged in today, so the check
           adapts rather than demanding a camera: with one online there must be image bytes,
           and with none there must be an explanation. Both are real answers; only silence
           would be a failure. */
        var inventory = (await registry.CallAsync("unifi__list_cameras", Empty())).Text;

        if (FirstOnlineCamera(inventory) is { Length: > 0 } name)
        {
            var seen = await registry.CallAsync("unifi__look_at_camera", Args($$"""{"camera":"{{name}}"}"""));

            Check($"looking through '{name}' returns a picture", seen.HasImage,
                  seen.HasImage ? $"{seen.Image!.Length / 1365} KB of {seen.ImageMediaType}" : Head(seen.Text));

            // The words matter as much as the picture: a brain with no eyes gets only these.
            Check("...and says in words what it looked through",
                  seen.Text.Contains(name, StringComparison.OrdinalIgnoreCase), Head(seen.Text));
        }
        else
        {
            Console.WriteLine("  ..     no camera is online, so the snapshot went untested");
        }

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

    /// The name of the first camera `list_cameras` reported as online, read out of the same
    /// prose a model reads. Parsing her own output rather than calling Protect again keeps
    /// the check honest: if the wording stops being readable, that is worth knowing too.
    private static string FirstOnlineCamera(string inventory) =>
        inventory.Split('\n')
                 .Where(line => line.Contains(" - online"))
                 .Select(line => line.Split(" (")[0].Trim())
                 .FirstOrDefault() ?? "";

    private static JsonElement Empty() => Args("{}");

    private static JsonElement Args(string json) => JsonDocument.Parse(json).RootElement.Clone();

    /// One line, so a failure is readable without a wall of network inventory behind it.
    private static string Head(string text) =>
        text.Split('\n')[0] is { Length: > 90 } long_ ? long_[..90] + "..." : text.Split('\n')[0];
}
```
