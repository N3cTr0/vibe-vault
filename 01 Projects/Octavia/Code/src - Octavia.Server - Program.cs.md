---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Program.cs
---

# src\Octavia.Server\Program.cs

```csharp
using System.Runtime.InteropServices;
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

        // Arrives seconds later, after everything above: her models load in the background so
        // the socket is reachable immediately. Printed when it lands rather than waited for.
        Report(being.Ears);

        return WaitForStop();
    }

    /// Says what her ears did, whenever they finish doing it.
    ///
    /// Un-awaited on purpose. Loading `large-v3-turbo` takes seconds, and a server that
    /// printed nothing until its models were ready would look hung during the one part of
    /// startup a person is actually watching — while the socket has been answering the whole
    /// time. The log records it either way.
    private static void Report(Task<string?> ears) => _ = ears.ContinueWith(finished =>
    {
        if (finished.IsFaulted)
        {
            Console.WriteLine($"  ears: would not open — {finished.Exception?.GetBaseException().Message}");
            return;
        }

        if (finished.Result is { } engine) Console.WriteLine($"  ears: {engine}");
    }, TaskScheduler.Default);

    /// Blocks until she is asked to stop, and lets `using` unwind on the way out.
    ///
    /// **Every way of asking has to arrive here**, which is more than Ctrl+C. Her MCP servers
    /// are child processes and her voice owns a sound card; since the ears now open at
    /// startup she also holds about 1.6 GB of speech model. A stop that skips `Dispose`
    /// leaves an orphaned `npx` behind and a server nobody can cleanly restart.
    ///
    /// `Console.CancelKeyPress` only ever covered Ctrl+C and Ctrl+Break. **Closing the
    /// console window — now the obvious thing to do, since there is a desktop shortcut that
    /// opens one — went down a different path entirely** and was never handled.
    /// `PosixSignalRegistration` is the one API that covers all of them: on Windows .NET maps
    /// SIGINT to Ctrl+C, SIGQUIT to Ctrl+Break and **SIGTERM to the window's close button**,
    /// and the same three lines mean the right thing if she ever runs on Linux.
    private static int WaitForStop()
    {
        var stopping = new ManualResetEventSlim(false);
        var unwound = new ManualResetEventSlim(false);

        void Stop(PosixSignalContext context)
        {
            // The runtime must not tear the process down where it stands, or Dispose never
            // runs. Windows still allows only a few seconds after a window close, so the
            // handler waits for the unwind rather than returning into a race with it.
            context.Cancel = true;
            Console.WriteLine("stopping...");
            stopping.Set();
            unwound.Wait(TimeSpan.FromSeconds(4));
        }

        using var interrupt = PosixSignalRegistration.Create(PosixSignal.SIGINT, Stop);
        using var quit = PosixSignalRegistration.Create(PosixSignal.SIGQUIT, Stop);
        using var terminate = PosixSignalRegistration.Create(PosixSignal.SIGTERM, Stop);

        stopping.Wait();

        // Logged before `using` unwinds `Being`, so the line is on disk even if tearing the
        // MCP servers down takes longer than Windows is prepared to wait.
        Log.Write("Octavia server stopped");
        unwound.Set();
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
