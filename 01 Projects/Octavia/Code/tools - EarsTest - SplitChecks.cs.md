---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\SplitChecks.cs
---

# tools\EarsTest\SplitChecks.cs

```csharp
// Stage 15 — she is a server, and the things that look at her are clients.
//
// The split is the kind of architecture that decays silently. Nothing stops somebody
// reaching for `new OctaviaSession()` in the client "just for this one thing", and the day
// after that there are two of her; nothing stops a `SaveFileDialog` going back into the
// session, and it would work perfectly on the machine it was written on and hang a headless
// server for ever.
//
// So the rules are checked as *text*, against the source rather than the build. That is
// blunt and it is the right instrument: these are statements about what may appear in which
// project, and a compiler cannot express them because all three assemblies legitimately see
// each other's internals.
internal static class SplitChecks
{
    private static int _failures;

    private static void Check(string name, bool ok, string detail = "")
    {
        Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}{(ok ? "" : $" — {detail}")}");
        if (!ok) _failures++;
    }

    public static int Run()
    {
        _failures = 0;
        Console.WriteLine("\nserver and clients:");

        var repo = Repo();
        if (repo is null)
        {
            Check("the repository is where the checks can see it", false,
                  "no Octavia.slnx above " + AppContext.BaseDirectory);
            return _failures;
        }

        var client = Read(Path.Combine(repo, "src", "Octavia.App"));
        var core = Read(Path.Combine(repo, "src", "Octavia.Core"));
        var page = Read(Path.Combine(repo, "src", "Octavia.Core", "wwwroot"), "*.js");

        Check("there is a client to check", client.Length > 0, "no .cs found under src/Octavia.App");
        Check("there is a core to check", core.Length > 0, "no .cs found under src/Octavia.Core");

        /* Acceptance 3. The client is a browser that knows where she lives — if it ever
           builds a session again it is not a client, it is a second Octavia, and the two
           would fight over her data folder, her port and her sound card. */
        foreach (var forbidden in new[] { "new OctaviaSession", "new WhisperRecognizer", "new ClaudeBrain", "new LocalBrain", "new KokoroVoice" })
            Check($"the client never does `{forbidden}`", !client.Contains(forbidden, StringComparison.Ordinal),
                  "the client has started containing her again");

        /* The fault that made the split necessary to get right rather than merely possible.
           A file dialog needs a dispatcher and somebody looking at it, and the session may
           now be running on a machine with neither — so the one control that exists for when
           she is broken would be broken by moving her. */
        foreach (var forbidden in new[] { "SaveFileDialog", "OpenFileDialog", "MessageBox", "Application.Current" })
            Check($"the core never reaches for `{forbidden}`", !core.Contains(forbidden, StringComparison.Ordinal),
                  "a headless server cannot show that, and would hang or throw where it did");

        /* One transport. The `postMessage` channel existed only for the WebView2 page hosted
           inside her own process; with that gone, code that still fell back to it would
           report success into a void — and it suppressed the "lost the connection" notice
           for exactly the face that now needs it most. */
        Check("the page has one way in and it is the socket",
              !page.Contains("chrome.webview", StringComparison.Ordinal),
              "a postMessage fallback is back, and nothing is listening at the other end");

        Check("the page reconnects when she goes away",
              page.Contains("scheduleReconnect", StringComparison.Ordinal),
              "a server restart would leave every face dark for ever");

        /* One version for three assemblies. `SystemReport.Version` reads whichever assembly
           is executing, so a per-project version would disagree in a diagnostics bundle
           before anybody noticed it had drifted. */
        var props = Path.Combine(repo, "Directory.Build.props");
        var shared = File.Exists(props) && File.ReadAllText(props).Contains("<Version>", StringComparison.Ordinal);
        Check("one version covers the whole of her", shared, $"no <Version> in {props}");

        var strays = Directory.Exists(Path.Combine(repo, "src"))
            ? Directory.EnumerateFiles(Path.Combine(repo, "src"), "*.csproj", SearchOption.AllDirectories)
                       .Where(f => File.ReadAllText(f).Contains("<Version>", StringComparison.Ordinal))
                       .Select(Path.GetFileName)
                       .ToArray()
            : [];

        Check("...and no project quietly keeps its own", strays.Length == 0, string.Join(", ", strays!));

        return _failures;
    }

    /// Every source file in a tree, concatenated. Crude, and it is the point: these checks
    /// ask whether a string appears anywhere in a project, which is exactly the shape of the
    /// rule being enforced.
    private static string Read(string dir, string pattern = "*.cs")
    {
        if (!Directory.Exists(dir)) return "";

        var text = new System.Text.StringBuilder();

        foreach (var file in Directory.EnumerateFiles(dir, pattern, SearchOption.AllDirectories))
        {
            // `obj` holds generated copies of things that are legitimately elsewhere, and
            // `bin` holds a copy of the page. Either would make these checks lie.
            if (file.Contains($"{Path.DirectorySeparatorChar}obj{Path.DirectorySeparatorChar}") ||
                file.Contains($"{Path.DirectorySeparatorChar}bin{Path.DirectorySeparatorChar}")) continue;

            text.Append(File.ReadAllText(file)).Append('\n');
        }

        return text.ToString();
    }

    /// The repository, found the same way `Paths` finds her data folder — by walking up for
    /// the solution file. A check that hardcoded `C:\Projects\Octavia` would pass on one
    /// machine and be a lie on every other.
    private static string? Repo()
    {
        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
            if (File.Exists(Path.Combine(dir.FullName, "Octavia.slnx")))
                return dir.FullName;

        return null;
    }
}
```
