---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\LocalServer.cs
---

# src\Octavia.App\LocalServer.cs

```csharp
using System.Diagnostics;
using System.Net;
using System.Net.Sockets;
using Octavia.Core;

namespace Octavia;

/// Starting her server, when the client is the only thing anybody double-clicked.
///
/// **This is Stage 15's bill, and it went unpaid for two versions.** Before the split
/// `Octavia.exe` *was* her, so starting it started everything. After it, a person who opens
/// her shortcut without the server already running waits thirty seconds and gets a grey
/// window — which is the desktop shortcut written the day before, doing exactly what it was
/// told.
///
/// **The reconnection in `bridge.js` cannot help here, and that is structural rather than an
/// oversight.** It recovers a socket that dropped; it cannot recover a page that was never
/// served, because the retry lives in a file that has to be downloaded from the thing that
/// is missing. A server that goes away is handled. A server that was never there is a
/// different fault and needs a different answer, and this is it.
///
/// So the client starts her when she is on this machine and is not already up.
///
/// **This is half of Stage 15 item 4, not a replacement for it.** That item is a Windows
/// Service *and* a client that starts it on demand, and this is the second half. It was
/// briefly argued here that the service was made unnecessary by the server and the client
/// always sharing a box — the owner corrected that, and rightly: *"it may not always be the
/// case."* The rule that settles where devices live is about a deployment, and a deployment
/// is the one thing in this project guaranteed to change. Code that assumes it will not is
/// exactly what item 3 exists to remove.
internal static class LocalServer
{
    internal enum Outcome
    {
        /// Something was already serving her. The usual case, and the quiet one.
        Answering,

        /// This client started her.
        Started,

        /// Her server is on another machine, so it is not this client's to start — and a
        /// client that started one anyway would be a second Octavia nobody asked for.
        Remote,

        /// Nothing is serving her, and there is no server beside this client to run.
        Missing,

        /// It was started and never opened its port.
        Failed
    }

    /// How long her server gets to bind before the client stops waiting.
    ///
    /// `Being.Start` opens the socket before it builds anything expensive — the ears warm
    /// afterwards, on their own task — so binding is the first thing she does and this is
    /// generous rather than tuned.
    private static readonly TimeSpan Grace = TimeSpan.FromSeconds(20);

    /// Kept only so the handle outlives the call that made it. Nothing asks it to stop — see
    /// the note below on why the client is not what stops her.
    private static Process? _ours;

    internal static Outcome Ensure(ClientConfig client, string? profile)
    {
        var (host, port) = Split(client.Authority);

        if (!IsLoopback(host))
        {
            Log.Write($"her server is at {host}, which is not this machine — not starting one");
            return Outcome.Remote;
        }

        if (Answers(port, TimeSpan.FromMilliseconds(500)))
        {
            Log.Write($"her server is already up on {port}");
            return Outcome.Answering;
        }

        if (FindServer() is not { } exe)
        {
            Log.Error($"nothing is serving her on {port}, and no Octavia.Server.exe was " +
                      $"found to start");
            return Outcome.Missing;
        }

        /* A registered service is the better thing to start, and is tried first.

           It outlives this client, it comes back after a reboot, and stopping it is a thing
           somebody can do deliberately from a shortcut — none of which is true of a console
           this window happened to spawn. `--start` exits non-zero when there is no service,
           which is a clean answer rather than something to probe for. */
        if (StartService(exe))
        {
            if (WaitForPort(port, Grace))
            {
                Log.Write("her service answered; attaching");
                return Outcome.Started;
            }

            Log.Warn("her service started and nothing answered on the port; trying a console");
        }

        /* Its own console, minimised.

           Hiding it outright would cost the shutdown path: on Windows a console's close
           button is what raises SIGTERM, and a server started with no console at all can
           only be killed — which skips the line that says she stopped, and that line is the
           whole of how a clean shutdown is told from a crash.

           A taskbar entry is also the honest thing to show. There really are two processes
           now, and pretending otherwise is how somebody ends up with an Octavia they cannot
           find and cannot stop. */
        var start = new ProcessStartInfo(exe)
        {
            UseShellExecute = true,
            WindowStyle = ProcessWindowStyle.Minimized,

            // Her own folder, not the client's. `Paths` resolves from the running exe rather
            // than the working directory so this changes nothing today — which is the reason
            // to set it now, while it is free, rather than after something starts reading a
            // relative path and inherits the wrong one.
            WorkingDirectory = Path.GetDirectoryName(exe)!
        };

        if (profile is { Length: > 0 })
        {
            start.ArgumentList.Add("--profile");
            start.ArgumentList.Add(profile);
        }

        Log.Write($"nothing on {port}; starting her server" +
                  (profile is { Length: > 0 } ? $" on the {profile} profile" : ""));

        try
        {
            _ours = Process.Start(start);
        }
        catch (Exception ex)
        {
            Log.Error($"her server would not start: {ex.Message}");
            return Outcome.Failed;
        }

        if (!WaitForPort(port, Grace))
        {
            Log.Error($"her server was started but nothing answered on {port} within " +
                      $"{Grace.TotalSeconds:0}s");
            return Outcome.Failed;
        }

        Log.Write("her server answered; attaching");
        return Outcome.Started;
    }
    /* **Why there is no `Stop` here, having written one and thrown it away.**

       The client can start her; it should not be what stops her, and three measured attempts
       are the argument rather than a preference.

       `CloseMainWindow` reports success, ends the process inside six seconds, and her
       shutdown handler never runs — nothing logged, her sound card and any MCP child
       released by the OS instead of by her. Closing the same console by hand logs the line
       correctly, so the two are not equivalent however much they look it.

       Ctrl+C is delivered and ignored: `AttachConsole` and `GenerateConsoleCtrlEvent` both
       return true and she carries on serving. A server started by a window does not answer
       to one.

       Ctrl+Break does work — it arrives as SIGQUIT and produced the clean shutdown line
       every time. **It also cannot be survived by whoever raises it.** The one-line way to
       deafen yourself, `SetConsoleCtrlHandler(NULL, TRUE)`, ignores Ctrl+C only; a real
       handler returning true did not save the caller either, which was measured twice from
       a test shell that exited 130 for its trouble. A client that takes itself down to stop
       her would skip its own `OnExit` and leave a dead tray icon behind — trading a tidy
       server for an untidy desktop.

       So she is stopped where stopping her is somebody's actual job: her own console's close
       button, which is the SIGTERM path and is proven, or the service in Stage 15 item 4,
       where the SCM does this properly and none of the above applies. **That item is not
       made unnecessary by the client being able to start her** — the two halves it describes
       are a service *and* a client that starts it on demand, and this is only the second.

       Leaving her running is also right on its own terms. A handset can be mid-conversation
       while nobody is at the desk, and closing a window on this machine is no reason to hang
       up on a phone in another room. */


    /// Whether a service is registered for her on this machine.
    ///
    /// Asked once, when the tray is built. `--service-status` exits 1 for "not installed"
    /// and 0 or 2 for a service that exists, which is the whole of the question here.
    internal static bool ServiceInstalled =>
        FindServer() is { } exe && Ask(exe, "--service-status") != 1;

    /// Starts or stops her service from the tray.
    ///
    /// Fire and forget, on a background thread, because both can take seconds — her ears
    /// alone are 1.6 GB — and a tray menu that freezes the desktop while it waits is worse
    /// than one that takes a moment to have an effect.
    internal static void ControlService(bool start) => Task.Run(() =>
    {
        if (FindServer() is not { } exe) return;

        var result = Ask(exe, start ? "--start" : "--stop");

        Log.Write(result == 0
            ? $"her service was {(start ? "started" : "stopped")} from the tray"
            : $"the tray could not {(start ? "start" : "stop")} her service (exit {result})");
    });

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
            Log.Warn($"could not run {Path.GetFileName(exe)} {argument}: {ex.Message}");
            return -1;
        }
    }

    /// Asks the server exe to start her service, and reports whether there was one.
    ///
    /// Shelling out rather than holding a `ServiceController` here on purpose: the SCM, the
    /// service name and the permissions it was installed with are the server's business,
    /// and a client that knew them would be a second place to keep them right.
    private static bool StartService(string exe)
    {
        var started = Ask(exe, "--start") == 0;
        if (started) Log.Write("her service was asked to start");
        return started;
    }

    /// Where her server is, in the two layouts that actually exist.
    ///
    /// `dist` is the shipped one and puts the pair side by side, which is what `README.md`
    /// describes and what gets copied to a machine. A working tree never does: every project
    /// has its own `bin`, so the two exes are cousins rather than siblings.
    ///
    /// **Both are handled, and the second is not indulgence.** A start path that only works
    /// after `dotnet publish` is a start path nobody exercises while writing the code that
    /// depends on it — which is exactly how this whole failure got shipped in the first
    /// place.
    private static string? FindServer()
    {
        var beside = Path.Combine(AppContext.BaseDirectory, "Octavia.Server.exe");
        if (File.Exists(beside)) return beside;

        var sep = Path.DirectorySeparatorChar;

        var sibling = Path.Combine(
            AppContext.BaseDirectory.Replace($"{sep}Octavia.App{sep}", $"{sep}Octavia.Server{sep}"),
            "Octavia.Server.exe");

        return File.Exists(sibling) ? sibling : null;
    }

    private static bool WaitForPort(int port, TimeSpan grace)
    {
        var until = DateTime.UtcNow + grace;

        while (DateTime.UtcNow < until)
        {
            // A server that fell over on the way up should be reported as that, rather than
            // as twenty seconds of nothing happening.
            if (_ours is { HasExited: true } dead)
            {
                Log.Error($"her server exited immediately, with code {dead.ExitCode}");
                return false;
            }

            if (Answers(port, TimeSpan.FromMilliseconds(300))) return true;
            Thread.Sleep(200);
        }

        return false;
    }

    /// A TCP connect and nothing more.
    ///
    /// Asking for her page would mean holding a credential and reasoning about status codes
    /// to answer a question that is really *is anything listening* — and the single-instance
    /// mutex in the server is what actually prevents two of her, so this only has to be right
    /// about the ordinary case.
    private static bool Answers(int port, TimeSpan timeout)
    {
        try
        {
            using var socket = new TcpClient();
            return socket.ConnectAsync(IPAddress.Loopback, port).Wait(timeout);
        }
        catch
        {
            // Refused is the answer, not an error: nothing is there.
            return false;
        }
    }

    private static bool IsLoopback(string host) =>
        host.Equals("localhost", StringComparison.OrdinalIgnoreCase) ||
        (IPAddress.TryParse(host.Trim('[', ']'), out var ip) && IPAddress.IsLoopback(ip));

    private static (string Host, int Port) Split(string authority)
    {
        var colon = authority.LastIndexOf(':');

        return colon > 0 && int.TryParse(authority[(colon + 1)..], out var port)
            ? (authority[..colon], port)
            : (authority, 8848);
    }
}
```
