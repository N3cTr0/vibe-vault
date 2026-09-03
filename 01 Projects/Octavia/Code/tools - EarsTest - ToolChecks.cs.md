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
                    Args = ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", script],
                    Secrets = ["HOUSE_PASSWORD"]
                }
            }
        };

        await using var registry = new ToolRegistry(config);
        await registry.StartAsync();

        Check("the server started", registry.Any);
        if (!registry.Any) return failures;

        var tools = await registry.ListAsync();
        Check("tools were listed", tools.Count == 4, $"{tools.Count}");

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

        /* **The risk heuristic reads words, and used to read them wrong.**

           `RiskOf` matched substrings, so a tool for reading the security log — described,
           accurately, as reporting intrusion attempts the gateway **blocked** — was classed as
           `Confirm`, because "blocked" contains "lock". That is not a needless question: a
           `Confirm` tool never runs without a person saying yes, and nothing asks on an hourly
           round, so the whole feature would silently have done nothing. A check found it; a
           person would have found a week of unexplained silence.

           It matches word *starts* now, so an ending is allowed and a beginning is not. Both
           halves are asserted, because the fix must not have made the guard blind. */
        var reading = Risk("recent_threats", "Read the security log: intrusion attempts the gateway detected and blocked.");
        Check("'blocked' is not 'lock'", reading == ToolRisk.Read, $"it is {reading}");

        Check("...nor is a clock a lock",
              Risk("get_clock", "Read the wall clock.") == ToolRisk.Read);

        Check("but a lock is still a lock",
              Risk("lock_door", "Lock the front door.") == ToolRisk.Confirm);

        Check("...and so is locking",
              Risk("secure_house", "Handles locking up at night.") == ToolRisk.Confirm);

        Check("...and unlocking",
              Risk("open_it", "Unlock the gate.") == ToolRisk.Confirm);

        /* `"arm "` used to be spelled with a trailing space to stop it matching "alarm" and
           "warm" — a hack that word-start matching makes unnecessary, and these are what say
           so. "disarm" is still caught because it is its own needle. */
        Check("a warm room is not arming anything",
              Risk("set_heat", "Make the room warm.") == ToolRisk.Act);

        Check("but arming is", Risk("secure", "Arm the system.") == ToolRisk.Confirm);
        Check("...and disarming is", Risk("unsecure", "Disarm the system.") == ToolRisk.Confirm);

        /* **A secret-shaped name is recognised, and an ordinary one is left alone.**

           This decides two things now: what a diagnostics bundle blanks on its way out of the
           building, and what the settings window refuses to draw. Over-redaction is not the
           safe default — it destroys the thing a bundle exists to carry — so both directions
           are asserted, including the two that a plain substring match got wrong. */
        foreach (var name in new[] { "UNIFI_API_KEY", "apiKey", "ApiKey", "Password", "AccessToken", "ClientSecret" })
            Check($"'{name}' reads as a secret", Sensitive.Looks(name));

        foreach (var name in new[] { "Hotkey", "UNIFI_HOST", "LocalModel", "Monkey" })
            Check($"'{name}' does not", !Sensitive.Looks(name));

        /* **`MaxTokens` reads as a secret, and that is not the bug it looks like.** The name
           really does contain the word "token", and nothing about the *name* can tell it apart
           from `AccessToken`. What tells them apart is the value: a budget is a number, a token
           is a string. So the bundle asks both questions and only ever redacts a string.

           Asserted in this direction on purpose, so that nobody reads a blanked `MaxTokens` as
           a flaw in the name test and special-cases it — which would quietly take `ApiKeys` and
           `SessionTokens` down with it. */
        Check("'MaxTokens' reads as a secret by name alone", Sensitive.Looks("MaxTokens"));

        /* **Sealing what somebody left in plain text.** The UniFi API key sat in `Env` for
           eighteen versions with a note saying it was "less than ideal", which is not a plan.
           `SealLoose` moves anything secret-shaped into the DPAPI store and takes it out of
           the file, and must leave everything else exactly where it was. */
        var loose = new OctaviaConfig
        {
            McpServers = new Dictionary<string, McpServer>
            {
                ["checkhouse"] = new()
                {
                    Command = pwsh,
                    Env = new Dictionary<string, string>
                    {
                        ["HOUSE_API_KEY"] = "a-secret-that-should-move",
                        ["HOUSE_HOST"] = "10.0.0.1"
                    }
                }
            }
        };

        /* **`SealLoose` saves**, and a check must never save over the real `config.json`.
           `OCTAVIA_CONFIG` exists for exactly this and is put back in the `finally`. */
        var realConfig = Environment.GetEnvironmentVariable("OCTAVIA_CONFIG");
        var scratchConfig = Path.Combine(Path.GetTempPath(), $"octavia-seal-{Guid.NewGuid():N}.json");
        Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", scratchConfig);

        try
        {
            var moved = SecretStore.SealLoose(loose);
            var house = loose.McpServers["checkhouse"];

            Check("a key in plain text is sealed", moved.Count == 1, string.Join(", ", moved));
            Check("...and taken out of the settings", !house.Env!.ContainsKey("HOUSE_API_KEY"),
                  "it is still in Env");
            Check("...and declared as a secret", house.Secrets?.Contains("HOUSE_API_KEY") == true,
                  string.Join(",", house.Secrets ?? []));
            Check("...and can be read back", SecretStore.ReadFor("checkhouse", "HOUSE_API_KEY") == "a-secret-that-should-move");
            Check("an ordinary setting is untouched", house.Env.TryGetValue("HOUSE_HOST", out var host) && host == "10.0.0.1",
                  "the host was moved or lost");

            // Running it twice must not double the `Secrets` entry or lose anything.
            Check("running it again does nothing", SecretStore.SealLoose(loose).Count == 0);
            Check("...and does not duplicate the declaration", house.Secrets!.Length == 1, $"{house.Secrets.Length}");
        }
        finally
        {
            SecretStore.ClearFor("checkhouse", "HOUSE_API_KEY");
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", realConfig);
            try { if (File.Exists(scratchConfig)) File.Delete(scratchConfig); } catch { /* a temp file */ }
        }

        /* **A sealed secret reaching the child process, without ever being in `config.json`.**

           The UniFi threat feed needs a password, and a password does not belong beside an
           API key in a file that is safe to read over somebody's shoulder. So a server may
           declare `Secrets`, and the values are filled at spawn from `SecretStore`.

           Checked in both directions, because the failure that matters is silent: a server
           handed nothing reports a login failure, which sends a person to check the account
           when the real answer is that nothing was ever stored — or that it was stored by a
           different Windows account, which DPAPI treats identically. The mock reports the
           *length* and never the value; a check that a secret arrived must not be a way to
           print one. */
        var before = (await registry.CallAsync("house__house_secret_check", stateArgs)).Text;
        Check("with nothing stored, no secret arrives", before.Contains("no secret"), before);

        SecretStore.WriteFor("house", "HOUSE_PASSWORD", "not-a-real-password");

        try
        {
            await using var sealedRegistry = new ToolRegistry(config);
            await sealedRegistry.StartAsync();

            var after = (await sealedRegistry.CallAsync("house__house_secret_check", stateArgs)).Text;
            Check("a stored secret reaches the server", after.Contains("secret arrived"), after);
            Check("...whole, and not truncated", after.Contains("19 characters"), after);
        }
        finally
        {
            // Never left behind: a test that stores a credential and forgets it is a test
            // that has changed the machine it ran on.
            SecretStore.ClearFor("house", "HOUSE_PASSWORD");
        }

        Check("...and is cleared away afterwards", !SecretStore.HasFor("house", "HOUSE_PASSWORD"));

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

        // Reads as the old `Grants` did, so every case below still says what it said.
        static bool Yes(Conversation.Consent? pending, string tool, string said) =>
            Conversation.Authorises(pending, tool, said) is not null;

        Check("a plain yes counts", Yes(Asked().TakeConsent(), door, "yes"));

        Check("so does go ahead", Yes(Asked().TakeConsent(), door, "go ahead"));

        Check("and a yes with words around it", Yes(Asked().TakeConsent(), door, "Yes, please do that."));

        Check("a no does not", !Yes(Asked().TakeConsent(), door, "no"));

        // The one a looser rule gets wrong: agreement and refusal in the same breath.
        Check("a yes with a caveat does not", !Yes(Asked().TakeConsent(), door, "yes, but not yet"));

        Check("silence on the subject does not",
              !Yes(Asked().TakeConsent(), door, "what is the weather like"));

        Check("a yes to one tool is not a yes to another",
              !Yes(Asked().TakeConsent(), "house__house_set_light", "yes"));

        // The lifetime: taking it clears it, so a second turn cannot reuse the first's yes.
        var once = Asked();
        var first = once.TakeConsent();
        var second = once.TakeConsent();
        Check("consent survives exactly one turn",
              Yes(first, door, "yes") && !Yes(second, door, "yes"));

        Check("and nothing is pending before she asks",
              !Yes(new Conversation().TakeConsent(), door, "yes"));

        /* **What a yes authorises is the call she described**, and this is the assertion the
           whole rule now rests on.

           Until v0.49.1 consent was matched against the arguments the model produced *after*
           the yes, which asks two independent generations to agree on a JSON object. They do
           not. Comparing them as JSON instead of as text (v0.49.0) fixed the spelling half —
           key order, whitespace — and could never have fixed this half, seen the same day on
           the local brain:

               asked:     set_port_power{"port":1,"on":false}
               after yes: set_port_power{"device":"unifi-gateway","port":1,"on":false}

           An argument added, refused, asked again, added again. The person says yes for ever.

           So the model is no longer asked to reproduce anything, and the call that runs is
           the one that was read out loud — which is also the only one they can be said to
           have agreed to. */
        var authorised = Conversation.Authorises(Asked().TakeConsent(), door, "yes");

        Check("a yes returns the call she asked about",
              authorised?.Arguments == back, authorised?.Arguments ?? "nothing");

        Check("...and it is the back door, whatever the model writes next",
              authorised?.Arguments != front);

        /* The safety this replaces is not weaker, it moved: the model reaching for the front
           door on its second attempt used to be caught by the comparison, and is now
           irrelevant, because the front door is never what runs. `SameCall` is still the
           test for *whether they differed* — the brains log it when they do, since a model
           rewriting a confirmed call is worth seeing. */
        Check("a different entity is still a different call",
              !Conversation.SameCall(back, front));

        const string asked = """{"device": "UDM", "port": 1}""";

        Check("key order is not a different call",
              Conversation.SameCall(asked, """{"port": 1, "device": "UDM"}"""),
              "the v0.49.0 half");

        Check("...nor is whitespace", Conversation.SameCall(asked, """{"device":"UDM","port":1}"""));

        Check("but a different port is", !Conversation.SameCall(asked, """{"port":2,"device":"UDM"}"""));

        Check("and an added argument is",
              !Conversation.SameCall(asked, """{"device":"UDM","port":1,"force":true}"""),
              "the v0.49.1 half — this is what looped");

        // Nested values and arrays reorder too, and a canonical form that only sorted the
        // top level would say two different calls were the same one.
        Check("nesting is compared the same way",
              Conversation.SameCall("""{"a":{"x":1,"y":2},"b":[1,2]}""", """{"b":[1,2],"a":{"y":2,"x":1}}"""));

        Check("but an array's order is part of it",
              !Conversation.SameCall("""{"b":[1,2]}""", """{"b":[2,1]}"""));

        // `off` and `on` are one boolean apart and cut power to different things.
        Check("switching off is not switching on",
              !Conversation.SameCall("""{"port":1,"on":false}""", """{"port":1,"on":true}"""));

        // Arguments that will not parse must not all collapse into one another.
        Check("unparsable arguments are not all the same call",
              !Conversation.SameCall("{not json", "{also not json"));

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

    /// The classifier on its own, with no server in the way. It reads a name and a description
    /// and nothing else, so it can be asked directly — which is the only way to test the
    /// wordings that are *not* in this repo, and those are most of them.
    private static ToolRisk Risk(string name, string description) =>
        McpClient.RiskOf(name, description);

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
