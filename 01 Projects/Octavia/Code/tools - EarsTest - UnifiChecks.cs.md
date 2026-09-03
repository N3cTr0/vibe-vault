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

        if (Configured() is not { } configured)
        {
            Console.WriteLine("  skipped: no 'unifi' server with a reachable key in her config");
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
                    Env = configured.Env,

                    // Whatever she seals, sealed the same way — so this exercises the real
                    // secret path rather than a copy of it, and the threat checks below skip
                    // by themselves when no password has been stored on this machine.
                    Secrets = [.. configured.Secrets]
                }
            }
        };

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        Check("the server started", registry.Any);
        if (!registry.Any) return failures;

        var tools = await registry.ListAsync();
        Check("all ten tools were listed", tools.Count == 10, $"{tools.Count}");
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
        string[] writes = ["unifi__power_cycle_port", "unifi__set_port_power"];

        var shouldRead = tools.Where(t => !writes.Contains(t.Name) && t.Risk != ToolRisk.Read).ToList();
        Check("every read is judged a read", shouldRead.Count == 0,
              string.Join(", ", shouldRead.Select(t => $"{t.Name} is {t.Risk}")));

        foreach (var written in writes)
        {
            var write = tools.FirstOrDefault(t => t.Name == written);
            Check($"{written} is judged a confirm", write?.Risk == ToolRisk.Confirm,
                  write is null ? "the tool is missing" : $"it is {write.Risk}");
        }

        /* **Neither of those classifications is a guess any more, and that is the point.**

           `set_port_power` came out `Confirm` on the heuristic — but only because the sentence
           pointing at the other tool contains the word "reboot". A description is prose, and
           prose gets edited; the tool would have gone quietly to `Act`, which she performs on
           her own, the first time somebody tidied that line. Both now say `destructiveHint`
           and the reads say `readOnlyHint`, so the wording carries the meaning and not the
           safety. The heuristic stays underneath for servers that annotate nothing. */
        var guessed = McpClient.RiskOf("set_port_power", "Switch the PoE power on one port off, and leave it off.");
        Check("the annotation is what makes the write a confirm, not the wording",
              guessed != ToolRisk.Confirm, $"the heuristic alone says {guessed}");

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

        /* `set_port_power` gets the same treatment, and for the same reason: every one of
           these stops before the PUT, so the suite exercises the guards without a camera
           anywhere losing power. The one path not checked here is the change itself, which
           is a probe — see `ToolLoopProbe`. */
        var offUnasked = (await registry.CallAsync(
            "unifi__set_port_power", Args("""{"port":4,"on":false}"""))).Text;

        Check("a port is not switched off without a yes",
              offUnasked.Contains("needs the person to say yes"), Head(offUnasked));

        var noSuchPort = (await registry.CallAsync(
            "unifi__set_port_power", Args("""{"port":99,"on":false}"""), confirmed: true)).Text;

        Check("...a port that does not exist is refused", noSuchPort.Contains("no port 99"), Head(noSuchPort));

        var notPoe = (await registry.CallAsync(
            "unifi__set_port_power", Args("""{"port":9,"on":false}"""), confirmed: true)).Text;

        Check("...and a port with no PoE has nothing to switch",
              notPoe.Contains("does not supply PoE"), Head(notPoe));

        /* Neither direction is assumed. A tool that took `on` as "anything present is true"
           would read a missing argument as *switch it on*, and one that defaulted the other
           way would read it as *cut the power* — so it refuses instead of picking. */
        var noDirection = (await registry.CallAsync(
            "unifi__set_port_power", Args("""{"port":4}"""), confirmed: true)).Text;

        Check("...and it will not guess on or off", noDirection.Contains("On, or off?"), Head(noDirection));

        /* The two writes must not be describable as each other. They are one word apart in
           English and very different in effect: one reboots a camera, the other leaves it
           dark until somebody says otherwise. The descriptions point at each other so the
           model can tell them apart, and that is what this pins. */
        var cycleTool = tools.First(t => t.Name == "unifi__power_cycle_port");
        var setTool = tools.First(t => t.Name == "unifi__set_port_power");

        Check("the cycle points at the switch for leaving a port off",
              cycleTool.Description.Contains("set_port_power"), Head(cycleTool.Description));

        Check("...and the switch points back for rebooting something",
              setTool.Description.Contains("power_cycle_port"), Head(setTool.Description));

        /* **The security log, which the API key reaches after all.**

           This block used to be wrapped in `if (SecretStore.HasFor("unifi", "UNIFI_PASSWORD"))`
           and headed "the API key cannot reach it at all" — which nobody had tried. The key
           reaches the legacy API perfectly well, so the login, the session, the CSRF token and
           a stored UniFi *account password* were all buying nothing. They are gone in v0.49.0
           and this runs unconditionally, which is the other half of the point: the skip meant
           the checks below only ran on a machine that had a password stored, so the day the
           password stopped being needed was a day this would have gone quiet rather than red. */
        {
            var counts = (await registry.CallAsync(
                "unifi__recent_threats", Args("""{"format":"counts"}"""))).Text;

            /* `total` is the line that is always there, including on an hour with nothing in
               it. `ThreatRound` treats its absence as *no answer* rather than as zero, which is
               the difference between a quiet hour and a gateway that has gone away — so the
               contract between the two files is asserted here rather than assumed. */
            Check("the security log answers in counts",
                  counts.Split('\n').Any(l => l.StartsWith("total\t", StringComparison.Ordinal)), Head(counts));

            Check("...and every line is a name and a number",
                  counts.Split('\n', StringSplitOptions.RemoveEmptyEntries)
                        .All(l => l.Split('\t') is [_, var n] && int.TryParse(n.Trim(), out _)),
                  Head(counts));

            var words = (await registry.CallAsync(
                "unifi__recent_threats", Args("""{"format":"words"}"""))).Text;

            Check("...and in words for her to read out",
                  words.Contains("security event", StringComparison.OrdinalIgnoreCase), Head(words));

            // Reading history is a read, whatever it is reading about. The word "threat" in a
            // description is exactly the sort of thing that could tip the heuristic.
            var log = tools.FirstOrDefault(t => t.Name == "unifi__recent_threats");
            Check("reading the security log is judged a read", log?.Risk == ToolRisk.Read,
                  log is null ? "the tool is missing" : $"it is {log.Risk}");
        }

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

    /// Her configured `unifi` server — its plain settings and the names of its sealed
    /// secrets — or null when there is not one to test.
    ///
    /// **It reads the config the way she reads it, which it did not used to.** This looked
    /// for `UNIFI_API_KEY` in `Env` and skipped the entire file when it was not there. In
    /// v0.48.0 the key was sealed and removed from `Env` — correctly — and every check in
    /// here quietly stopped running, reporting *"skipped: no 'unifi' server with a key"* on
    /// the one machine that has both. A skip reads like a machine that was never set up, so
    /// nothing about it looked wrong.
    ///
    /// Taking `Secrets` and letting `ToolRegistry` fill them is also the more honest test:
    /// it is the path production uses, so it fails if sealing itself breaks.
    private static (Dictionary<string, string> Env, List<string> Secrets)? Configured()
    {
        try
        {
            var file = Path.Combine(Paths.DataDir, "config.json");
            if (!File.Exists(file)) return null;

            using var json = JsonDocument.Parse(File.ReadAllText(file));

            if (!json.RootElement.TryGetProperty("McpServers", out var servers) ||
                !servers.TryGetProperty("unifi", out var unifi))
                return null;

            var env = unifi.TryGetProperty("Env", out var e) && e.ValueKind == JsonValueKind.Object
                ? e.EnumerateObject().ToDictionary(p => p.Name, p => p.Value.GetString() ?? "")
                : [];

            var secrets = unifi.TryGetProperty("Secrets", out var s) && s.ValueKind == JsonValueKind.Array
                ? s.EnumerateArray().Select(v => v.GetString() ?? "").Where(v => v.Length > 0).ToList()
                : [];

            // The key has to be reachable one way or the other, and sealed is now the normal
            // way. Without it the server starts and every call answers "UNIFI_API_KEY is not
            // set", which would be nine failures describing one missing credential.
            var haveKey = (env.TryGetValue("UNIFI_API_KEY", out var plain) && plain.Length > 0) ||
                          SecretStore.ReadFor("unifi", "UNIFI_API_KEY") is { Length: > 0 };

            return haveKey ? (env, secrets) : null;
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
