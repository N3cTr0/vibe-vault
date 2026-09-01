---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Program.cs
---

# src\Octavia.Server\Program.cs

```csharp
using Octavia.Core;
using Octavia.Diagnostics;

namespace Octavia.Server;

/// The whole of a server: load her settings, open the socket, and stay out of the way.
///
/// Everything a renderer needs already travels over that socket, so there is nothing here
/// that the window used to do and this cannot. What the window has that this does not is a
/// screen — see `Being` for why that turns out to be the only difference.
internal static class Program
{
    private const string MutexName = @"Local\Octavia.Server.SingleInstance";

    private static int Main(string[] args)
    {
        // The console's default code page is not UTF-8, and her prose has em-dashes in it.
        // Without this the first thing a new server prints is mojibake, which is a poor
        // advertisement for the care taken everywhere else.
        try { Console.OutputEncoding = System.Text.Encoding.UTF8; } catch (IOException) { }

        var profile = ValueArgument(args, "--profile", "-p");

        // Before the single-instance check, deliberately: the one command that has to work
        // when she is already running *and* when she is too broken to start.
        if (ValueArgument(args, "--diagnostics") is { } bundlePath)
            return WriteBundle(bundlePath, profile);

        using var onlyOne = new Mutex(true, MutexName, out var isFirst);
        if (!isFirst)
        {
            Console.Error.WriteLine("Octavia's server is already running.");
            return 1;
        }

        WatchForCrashes();

        var config = OctaviaConfig.Load(profile);
        Log.Write($"Octavia server starting {SystemReport.Version}; data in {Paths.DataDir}");

        using var being = Being.Start(config);

        if (being is null)
        {
            // Fatal here where the window survives it. A window that cannot bind still has
            // its own page; a server that cannot bind has no way for anything to reach her
            // at all, and a process that looks alive and answers nobody is worse than one
            // that stopped and said why.
            var message = $"nothing could bind port {config.FacePort}, so no face could ever reach her";
            Log.Error($"server: {message}");
            Console.Error.WriteLine($"Octavia's server could not start: {message}");
            return 1;
        }

        Console.WriteLine($"Octavia {SystemReport.Version} — {config.Profile} profile, {config.Brain} brain");
        Console.WriteLine($"  a face may attach at {being.Address}");
        if (config.RemoteAccess) Console.WriteLine("  remote faces may attach with the remote key");
        Console.WriteLine("  ctrl+c to stop her");

        return WaitForStop();
    }

    /// Blocks until Ctrl+C, and lets `using` unwind on the way out.
    ///
    /// `Cancel = true` so the runtime does *not* kill the process where it stands: her MCP
    /// servers are child processes and her voice owns a sound card, and both are released in
    /// `Dispose`. A server that leaves an orphaned `npx` behind every time it stops is a
    /// server nobody can restart cleanly.
    private static int WaitForStop()
    {
        var stopping = new ManualResetEventSlim(false);

        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true;
            Console.WriteLine("stopping...");
            stopping.Set();
        };

        // A service, a scheduled task or a terminal being closed asks this way instead.
        AppDomain.CurrentDomain.ProcessExit += (_, _) => stopping.Set();

        stopping.Wait();
        Log.Write("Octavia server stopped");
        return 0;
    }

    /// Writes a bundle with no session at all — the one thing still possible when she will
    /// not start, because the environment, the settings and the log are exactly what
    /// explains why.
    private static int WriteBundle(string path, string? profile)
    {
        try
        {
            var config = OctaviaConfig.Load(profile);
            DiagnosticsBundle.WriteAsync(path, config, HostSnapshot.Stopped).GetAwaiter().GetResult();
            Console.WriteLine($"Diagnostics written to {path}");
            return 0;
        }
        catch (Exception ex)
        {
            Log.Error("headless diagnostics failed", ex);
            Console.Error.WriteLine($"Could not write diagnostics: {ex.Message}");
            return 1;
        }
    }

    /// Two ways she can fail without anyone finding out. There is no UI thread here, so the
    /// third — the dispatcher's — has no counterpart; the client keeps that one.
    private static void WatchForCrashes()
    {
        AppDomain.CurrentDomain.UnhandledException += (_, args) =>
        {
            if (args.ExceptionObject is Exception ex) Log.Error("unhandled on a background thread", ex);
            else Log.Error($"unhandled on a background thread: {args.ExceptionObject}");
        };

        // A faulted task nobody awaited. Usually harmless, occasionally the only trace of a
        // subsystem that quietly stopped working.
        TaskScheduler.UnobservedTaskException += (_, args) =>
        {
            Log.Warn($"unobserved task exception: {args.Exception.Message}");
            args.SetObserved();
        };
    }

    /// --name value, --name=value, or an alias such as -p.
    private static string? ValueArgument(string[] args, string name, string? alias = null)
    {
        for (var i = 0; i < args.Length; i++)
        {
            var arg = args[i];
            if (arg.StartsWith(name + "=", StringComparison.OrdinalIgnoreCase))
                return arg[(name.Length + 1)..];

            var named = arg.Equals(name, StringComparison.OrdinalIgnoreCase)
                     || (alias is not null && arg.Equals(alias, StringComparison.OrdinalIgnoreCase));
            if (named && i + 1 < args.Length) return args[i + 1];
        }

        return null;
    }
}
```
