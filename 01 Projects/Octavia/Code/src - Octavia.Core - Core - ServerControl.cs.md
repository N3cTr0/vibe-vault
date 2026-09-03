---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\ServerControl.cs
---

# src\Octavia.Core\Core\ServerControl.cs

```csharp
using System.Diagnostics;

namespace Octavia.Core;

/// Finding her server, and asking it to start or stop.
///
/// **Lifted out of the client in v0.46.0 rather than copied**, when the tray moved to a
/// process of its own. Two copies of "where is `Octavia.Server.exe`" is two answers the day
/// somebody rearranges the folders, and this one is already subtle: a published `dist` puts
/// every exe side by side, and a working tree gives each project its own `bin`, so the pair
/// are cousins rather than siblings.
///
/// Nothing here talks to the service control manager directly. The server owns the service
/// name, the account it logs on as and the rights it was installed with; a second process
/// that knew those would be a second place to keep them right.
internal static class ServerControl
{
    private const string Exe = "Octavia.Server.exe";

    /// Her server, in the two layouts that exist, or null.
    ///
    /// **Both are handled, and the second is not indulgence.** A path that only works after
    /// `dotnet publish` is a path nobody exercises while writing the code that depends on it,
    /// which is exactly how the client once shipped unable to start her at all.
    public static string? Find()
    {
        var beside = Path.Combine(AppContext.BaseDirectory, Exe);
        if (File.Exists(beside)) return beside;

        // A working tree: walk up to the repository and across into the server's own bin.
        var here = new DirectoryInfo(AppContext.BaseDirectory);

        while (here is not null)
        {
            var bin = Path.Combine(here.FullName, "src", "Octavia.Server", "bin");

            if (Directory.Exists(bin))
            {
                var found = Directory.EnumerateFiles(bin, Exe, SearchOption.AllDirectories)
                                     .OrderByDescending(File.GetLastWriteTimeUtc)
                                     .FirstOrDefault();
                if (found is not null) return found;
            }

            here = here.Parent;
        }

        return null;
    }

    /// Whether a service is registered. `--service-status` answers 1 for "not installed",
    /// which is a clean answer rather than something to probe the registry for.
    public static bool ServiceInstalled => Find() is { } exe && Ask(exe, "--service-status") != 1;

    /// `0` running, `1` not installed, `2` stopped — and `-1` when the server exe is missing,
    /// which is a fourth state and must not be folded into "stopped".
    public static int ServiceState => Find() is { } exe ? Ask(exe, "--service-status") : -1;

    /// Starts or stops her service, and reports whether it worked.
    ///
    /// Synchronous: a caller that wants this off the UI thread can say so, and one that wants
    /// to know the answer — a tray menu that then re-reads the state — cannot get it from a
    /// fire-and-forget.
    public static bool Control(bool start)
    {
        if (Find() is not { } exe) return false;

        var result = Ask(exe, start ? "--start" : "--stop");

        Log.Write(result == 0
            ? $"her service was {(start ? "started" : "stopped")}"
            : $"could not {(start ? "start" : "stop")} her service (exit {result})");

        return result == 0;
    }

    /// Stop, then start. **Not one command**: the service control manager returns from a stop
    /// when the service says it has stopped, and her unwind — three MCP child processes, a
    /// voice engine, 1.6 GB of Whisper — finishes after that. Starting immediately can find
    /// the port still held, which reads as "she would not start" and is really "she had not
    /// finished stopping".
    public static bool Restart()
    {
        if (!Control(start: false)) return false;

        for (var waited = 0; waited < 20; waited++)
        {
            if (ServiceState == 2) break;
            Thread.Sleep(500);
        }

        return Control(start: true);
    }

    /// Runs the server exe with one switch and hands back its exit code.
    private static int Ask(string exe, string argument)
    {
        try
        {
            using var asked = Process.Start(new ProcessStartInfo(exe, argument)
            {
                UseShellExecute = false,
                CreateNoWindow = true,
                RedirectStandardOutput = true,
                RedirectStandardError = true
            });

            if (asked is null) return -1;

            asked.WaitForExit(40_000);
            return asked.ExitCode;
        }
        catch (Exception ex)
        {
            Log.Warn($"could not ask the server '{argument}': {ex.Message}");
            return -1;
        }
    }
}
```
