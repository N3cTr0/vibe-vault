---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Service.cs
---

# src\Octavia.Server\Service.cs

```csharp
using System.Diagnostics;
using System.Security.Principal;
using System.ServiceProcess;
using Octavia.Core;
using Octavia.Diagnostics;

namespace Octavia.Server;

/// Her, as a Windows service — Stage 15 item 4.
///
/// **The console stayed the default and this is the addition**, which is the order the
/// roadmap argued for: a service that fails before its first log line is diagnosed by
/// guesswork, so the boring version had to work first. It did, for four versions.
///
/// **Service mode is asked for explicitly**, with `--service`, rather than sniffed from
/// `Environment.UserInteractive`. A guess that is wrong produces a process which neither
/// runs as a service nor prints anything, and the flag is written into the registered
/// command line once, at install, where it can be read back with `sc qc`.
internal sealed class Service : ServiceBase
{
    internal const string Name = "Octavia";

    private readonly string? _profile;
    private Being? _being;

    internal Service(string? profile)
    {
        _profile = profile;
        ServiceName = Name;
        CanStop = true;
        CanShutdown = true;
    }

    protected override void OnStart(string[] args)
    {
        var config = OctaviaConfig.Load(_profile);
        Log.Write($"Octavia service starting {SystemReport.Version}; data in {Paths.DataDir}");

        _being = Being.Start(config);

        if (_being is null)
        {
            /* The same judgement the console makes, for the same reason: a server that
               cannot bind can never be reached, and one that looks alive and answers nobody
               is worse than one that stopped and said why.

               Reporting a non-zero ExitCode is what makes the SCM show this as a failure
               rather than as a service that started and mysteriously went away — and it is
               how `Recovery` actions get a chance to fire. */
            Log.Error($"service: nothing could bind port {config.FacePort}");
            ExitCode = 1067;   // ERROR_PROCESS_ABORTED, which the SCM renders sensibly
            Stop();
            return;
        }

        Log.Write($"service: a face may attach at {_being.Address}");
        Report(_being.Ears);
    }

    protected override void OnStop() => Unwind();

    // A machine going down is a stop with a shorter deadline, not a different event.
    protected override void OnShutdown() => Unwind();

    private void Unwind()
    {
        if (_being is null) return;

        _being.Dispose();
        _being = null;

        // The same line the console writes, deliberately: "did she stop cleanly" should be
        // one question with one answer, whichever way she was started.
        Log.Write("Octavia server stopped");
    }

    private static void Report(Task<string?> ears) => _ = ears.ContinueWith(finished =>
    {
        if (finished.IsFaulted)
            Log.Warn($"service: ears would not open — {finished.Exception?.GetBaseException().Message}");
        else if (finished.Result is { } engine)
            Log.Write($"service: ears ready: {engine}");
    }, TaskScheduler.Default);

    // ---- registering, and the desktop's start and stop --------------------------------

    /// Installs the service, points it at this exe, and hands the current user the right to
    /// start and stop it without becoming an administrator first.
    ///
    /// That last part is the whole point of the request this answers. A service is normally
    /// an administrator's object: every start and stop raises a UAC prompt, which is a poor
    /// thing to put between somebody and their own companion. One `sdset` at install time
    /// removes it permanently, for this service only, for this account only.
    internal static int Install(string? profile)
    {
        if (Relaunch("--install", profile) is { } relaunched) return relaunched;

        var exe = Environment.ProcessPath;
        if (exe is null)
        {
            Console.Error.WriteLine("could not work out where this exe lives.");
            return 1;
        }

        /* The exe path is quoted *inside* the value, and the whole thing is passed as one
           argument rather than inside a command string.

           Both halves matter. Windows splits `ImagePath` on spaces to find the executable,
           so an unquoted `C:\Program Files\Octavia\Octavia.Server.exe` resolves to
           `C:\Program.exe` and the service simply never starts — the classic unquoted
           service path. And building the whole `sc` command as one string leaves the
           quoting to two layers of parsing that disagree: it was measured stripping them,
           which is invisible on a path that happens to have no spaces in it. */
        var command = profile is { Length: > 0 }
            ? $"\"{exe}\" --service --profile {profile}"
            : $"\"{exe}\" --service";

        if (Sc(["create", Name, "binPath=", command, "DisplayName=", "Octavia", "start=", "auto"])
            is var created && created != 0)
        {
            Console.Error.WriteLine("could not register the service. Is it already installed?");
            return created;
        }

        Sc(["description", Name,
            "Octavia's server: her brain, ears, voice and the socket faces attach to."]);

        if (GrantControl() is var granted && granted != 0)
            Console.WriteLine("registered, but this account still needs administrator rights to " +
                              "start and stop her. See the log.");

        Console.WriteLine($"Octavia is registered as a service, and starts with Windows.");
        Console.WriteLine($"  start it   {Path.GetFileName(exe)} --start");
        Console.WriteLine($"  stop it    {Path.GetFileName(exe)} --stop");
        Console.WriteLine();

        /* Said at install rather than discovered later, because the failure it causes is
           silent and looks like a broken brain rather than like a permissions problem: her
           key is DPAPI-sealed to *a user account*, and a service installed this way logs on
           as LocalSystem, which is not that account.

           **The fix is measured, not assumed.** Setting the service to log on as the user
           was tried: her key decrypts, the hosted brain answers, and the audio devices still
           open from session 0. So this recommends the thing that is known to work rather
           than listing two possibilities and leaving the reader to find out. */
        Console.WriteLine("Note: a service logs on as LocalSystem by default, and her API key is");
        Console.WriteLine("sealed to *your* Windows account — so the hosted brain will not find it.");
        Console.WriteLine("The local brain, which is the default, is unaffected.");
        Console.WriteLine();
        Console.WriteLine("To use Claude from the service, set it to log on as your own account:");
        Console.WriteLine("  services.msc -> Octavia -> Log On -> This account.");
        Console.WriteLine("That is verified working, session 0 and all. A machine-wide");
        Console.WriteLine("ANTHROPIC_API_KEY also works, and needs no password.");

        return 0;
    }

    internal static int Uninstall()
    {
        if (Relaunch("--uninstall", null) is { } relaunched) return relaunched;

        Stop(quiet: true);

        var removed = Sc(["delete", Name]);
        Console.WriteLine(removed == 0
            ? "Octavia's service is gone. The console still works as it always did."
            : "could not remove the service; is it installed?");

        return removed;
    }

    /// Whether the service exists, and what it is doing.
    internal static ServiceControllerStatus? Status()
    {
        try
        {
            using var service = new ServiceController(Name);
            return service.Status;
        }
        catch (InvalidOperationException)
        {
            // Not installed. An expected answer rather than a fault: everything that asks
            // this question has a perfectly good path for "there is no service".
            return null;
        }
    }

    internal static int Start()
    {
        if (Status() is not { } status)
        {
            Console.Error.WriteLine("Octavia is not installed as a service. Run --install first.");
            return 1;
        }

        if (status is ServiceControllerStatus.Running or ServiceControllerStatus.StartPending)
        {
            Console.WriteLine("Octavia's service is already running.");
            return 0;
        }

        try
        {
            using var service = new ServiceController(Name);
            service.Start();
            service.WaitForStatus(ServiceControllerStatus.Running, TimeSpan.FromSeconds(30));
            Console.WriteLine("Octavia is running.");
            return 0;
        }
        catch (Exception ex)
        {
            // The interesting case, and the one the grant exists to prevent: access denied
            // because this account was never given the right to start it.
            Console.Error.WriteLine($"could not start her: {ex.Message}");
            return 1;
        }
    }

    internal static int Stop(bool quiet = false)
    {
        if (Status() is not { } status)
        {
            if (!quiet) Console.Error.WriteLine("Octavia is not installed as a service.");
            return quiet ? 0 : 1;
        }

        if (status is ServiceControllerStatus.Stopped or ServiceControllerStatus.StopPending)
        {
            if (!quiet) Console.WriteLine("Octavia's service is already stopped.");
            return 0;
        }

        try
        {
            using var service = new ServiceController(Name);
            service.Stop();
            service.WaitForStatus(ServiceControllerStatus.Stopped, TimeSpan.FromSeconds(30));
            if (!quiet) Console.WriteLine("Octavia has stopped.");
            return 0;
        }
        catch (Exception ex)
        {
            if (!quiet) Console.Error.WriteLine($"could not stop her: {ex.Message}");
            return 1;
        }
    }

    /// Adds one allow-entry to the service's own security descriptor: this account may
    /// start, stop, pause and query it, and nothing else.
    ///
    /// Read the descriptor rather than writing a whole new one. A hand-written SDDL that
    /// happens to omit an entry Windows put there is how a service becomes unmanageable by
    /// the system that installed it — so this splices one ACE into what is already there.
    private static int GrantControl()
    {
        var sid = WindowsIdentity.GetCurrent().User?.Value;
        if (sid is null)
        {
            Log.Warn("service: no SID for this account; leaving the default permissions alone");
            return 1;
        }

        var current = ScOutput(["sdshow", Name]).Trim();
        var start = current.IndexOf("D:", StringComparison.Ordinal);
        if (start < 0)
        {
            Log.Warn($"service: could not read the security descriptor ('{current}')");
            return 1;
        }

        var firstEntry = current.IndexOf('(', start);
        if (firstEntry < 0)
        {
            Log.Warn("service: the security descriptor had no entries to splice into");
            return 1;
        }

        /* RP start, WP stop, DT pause and continue, LO interrogate, CR user-defined
           control, RC read the descriptor. Not WD or WO: the right to hand *other* people
           control of her is an administrator's to give, and stays there. */
        var granted = current[..firstEntry] + $"(A;;RPWPDTLOCRRC;;;{sid})" + current[firstEntry..];

        var result = Sc(["sdset", Name, granted]);
        if (result == 0) Log.Write("service: this account may start and stop her without elevation");
        else Log.Warn($"service: sdset failed ({result}); starting her will ask for administrator");

        return result;
    }

    /// Runs the whole command again as an administrator, or null when it already is one.
    ///
    /// Registering a service is genuinely an administrator's job, so the alternative to
    /// this is an error message telling somebody to find their terminal and start again.
    /// The prompt is Windows' own, and it names the exe being elevated.
    private static int? Relaunch(string verb, string? profile)
    {
        if (new WindowsPrincipal(WindowsIdentity.GetCurrent()).IsInRole(WindowsBuiltInRole.Administrator))
            return null;

        var exe = Environment.ProcessPath;
        if (exe is null) return null;

        var arguments = profile is { Length: > 0 } ? $"{verb} --profile {profile}" : verb;

        Console.WriteLine($"{verb} needs administrator rights; asking Windows for them...");

        try
        {
            var elevated = Process.Start(new ProcessStartInfo(exe, arguments)
            {
                UseShellExecute = true,
                Verb = "runas"
            });

            elevated?.WaitForExit();
            return elevated?.ExitCode ?? 1;
        }
        catch (Exception ex)
        {
            // Cancelling the UAC prompt lands here, and is a choice rather than a fault.
            Console.Error.WriteLine($"not elevated: {ex.Message}");
            return 1;
        }
    }

    private static int Sc(string[] arguments) => Run(arguments, out _);

    private static string ScOutput(string[] arguments)
    {
        Run(arguments, out var output);
        return output;
    }

    /// `sc.exe`, with every argument passed separately.
    ///
    /// `ArgumentList` rather than a command string, so the runtime does the escaping once
    /// and correctly. The string form looks tidier and quietly loses quotes a service path
    /// depends on.
    private static int Run(string[] arguments, out string output)
    {
        output = "";

        var start = new ProcessStartInfo("sc.exe")
        {
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        };

        foreach (var argument in arguments) start.ArgumentList.Add(argument);

        try
        {
            using var process = Process.Start(start);
            if (process is null) return 1;

            output = process.StandardOutput.ReadToEnd();
            process.WaitForExit(30_000);
            return process.ExitCode;
        }
        catch (Exception ex)
        {
            Log.Warn($"service: sc {arguments.FirstOrDefault()} failed: {ex.Message}");
            return 1;
        }
    }
}
```
