---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Program.cs
---

# src\Octavia.Server\Program.cs

```csharp
using System.Runtime.InteropServices;
using System.ServiceProcess;
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

        /* Managing the service is also outside the check, and for a plainer reason: every
           one of these is *about* a running server, so refusing them because one is running
           would be exactly backwards. */
        // Also outside the check: storing a secret is something you do *because* she is
        // running badly, and refusing it while she is up would be the wrong way round.
        if (ValueArgument(args, "--secret") is { } secret) return StoreSecret(secret);

        if (Has(args, "--install")) return Service.Install(profile);
        if (Has(args, "--uninstall")) return Service.Uninstall();
        if (Has(args, "--start")) return Service.Start();
        if (Has(args, "--stop")) return Service.Stop();
        if (Has(args, "--service-status")) return ReportService();

        /* Service mode, which never returns until the SCM says so.
           Also outside the mutex, and this is the subtle one: a service runs in session 0
           and a console runs in the user's, so a `Local\` mutex cannot see across them and
           would pass in both. The real guard is the port — whichever starts second cannot
           bind and says so. See the message below. */
        if (Has(args, "--service"))
        {
            ServiceBase.Run(new Service(profile));
            return 0;
        }

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

            // The likeliest cause once she can be a service, and invisible from here: the
            // service is in another session, so nothing above this line could have noticed.
            if (Service.Status() is System.ServiceProcess.ServiceControllerStatus.Running)
                Console.Error.WriteLine("Her service is running and already holds that port. " +
                                        "Stop it first: Octavia.Server.exe --stop");

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

    /// A switch with no value of its own.
    private static bool Has(string[] args, string name) =>
        args.Any(a => a.Equals(name, StringComparison.OrdinalIgnoreCase));

    /// Stores one secret a tool server needs, sealed to this Windows account.
    ///
    /// **It is typed here and nowhere else.** Not into `config.json`, which sits in plain text
    /// beside things that are safe to read over a shoulder; not onto a command line, which
    /// every account on the machine can see in the process list; and not into her Settings
    /// panel, which would put it on the wire. Read with the echo off, sealed, and forgotten.
    ///
    ///     Octavia.Server.exe --secret unifi:UNIFI_PASSWORD
    ///
    /// The name is the same pair that appears in `McpServer.Secrets`, so there is one
    /// spelling of it and not two.
    private static int StoreSecret(string name)
    {
        var parts = name.Split(':', 2);

        if (parts.Length != 2 || parts.Any(string.IsNullOrWhiteSpace))
        {
            Console.Error.WriteLine("Usage: --secret <server>:<VARIABLE>, for example unifi:UNIFI_PASSWORD");
            return 2;
        }

        var (server, variable) = (parts[0].Trim(), parts[1].Trim());

        /* Three tries, because the first one is the one that goes wrong.
           Launching this command *is* a keypress, and if the Enter that ran it is still in
           the input buffer when the read starts, it is taken as an empty answer and the whole
           thing exits saying "nothing entered" before the person has typed a character. The
           buffer is drained below; the retry is what makes that recoverable rather than
           mystifying if anything else ever does the same. */
        string? value = null;

        for (var attempt = 1; attempt <= 3 && string.IsNullOrEmpty(value); attempt++)
        {
            Console.Write($"Value for {variable} on '{server}' (typing is hidden, Escape to cancel): ");
            value = ReadHidden();
            Console.WriteLine();

            if (value is null)
            {
                Console.WriteLine("Cancelled; nothing changed.");
                return 1;
            }

            if (value.Length == 0 && attempt < 3) Console.WriteLine("Nothing was typed. Try again.");
        }

        if (string.IsNullOrEmpty(value))
        {
            Console.WriteLine("Nothing entered; nothing changed.");
            return 1;
        }

        try
        {
            SecretStore.WriteFor(server, variable, value);
        }
        catch (Exception ex)
        {
            Console.Error.WriteLine($"Could not store it: {ex.Message}");
            return 3;
        }

        Console.WriteLine($"Stored, sealed to {Environment.UserName} on this machine.");
        Console.WriteLine($"Add \"Secrets\": [\"{variable}\"] to the '{server}' entry in config.json if it is not there,");
        Console.WriteLine("then restart her. The value is never written to config.json or to her log.");

        /* The one piece of advice that stops the commonest failure. DPAPI seals to an
           account, so a secret stored by a person and read back by a service running as
           LocalSystem is unopenable — indistinguishable, from the server's side, from never
           having been stored at all. README says the same about the API key. */
        Console.WriteLine();
        Console.WriteLine("If her service logs on as LocalSystem rather than as you, it will not be able");
        Console.WriteLine("to open this. services.msc -> Octavia -> Log On -> This account.");

        return 0;
    }

    /// Reads a secret without echoing it. Null means cancelled; empty means nothing typed.
    private static string? ReadHidden()
    {
        /* **`Console.ReadKey` throws outright when input is redirected**, and "does not have a
           console" covers more than it sounds like: a Run button in an editor, a pipeline, a
           scheduled task. The first version of this crashed with a stack trace in exactly that
           case, which is a poor way to learn that the password was never stored.

           A redirected stream is read as a line instead. It is warned about rather than
           quietly accepted, because a secret arriving down a pipe came from somewhere — a
           script, a shell history — and that somewhere now has it. */
        if (Console.IsInputRedirected)
        {
            Console.WriteLine();
            Console.WriteLine("Input is redirected, so the typing cannot be hidden. Whatever fed this");
            Console.WriteLine("(a shell history, a script) now holds the secret. Prefer a real terminal.");
            return Console.ReadLine()?.Trim();
        }

        /* Running this command means pressing Enter, and that keypress can still be sitting in
           the buffer when the first read happens — taken as an empty answer, reported as
           "nothing entered", before a character has been typed. */
        while (Console.KeyAvailable) Console.ReadKey(intercept: true);

        var typed = new System.Text.StringBuilder();

        while (true)
        {
            var key = Console.ReadKey(intercept: true);

            if (key.Key == ConsoleKey.Enter) return typed.ToString();
            if (key.Key == ConsoleKey.Escape) return null;

            if (key.Key == ConsoleKey.Backspace)
            {
                if (typed.Length > 0) typed.Length--;
                continue;
            }

            // Control characters are not part of a password and would otherwise be stored.
            if (!char.IsControl(key.KeyChar)) typed.Append(key.KeyChar);
        }
    }

    /// What the service is doing, in one line, for a person rather than a script.
    private static int ReportService()
    {
        switch (Service.Status())
        {
            case null:
                Console.WriteLine("Octavia is not installed as a service.");
                return 1;

            case var status:
                Console.WriteLine($"Octavia's service is {status.ToString()!.ToLowerInvariant()}.");
                return status == ServiceControllerStatus.Running ? 0 : 2;
        }
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
