---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\App.xaml.cs
---

# src\Octavia.App\App.xaml.cs

```csharp
using System.Drawing;
using System.Windows;
using System.Windows.Forms;
using Octavia.Core;
using Octavia.Diagnostics;
using Application = System.Windows.Application;

namespace Octavia;

/// The client's shell: one window, one tray icon, one instance.
///
/// **What is no longer here is the point of Stage 15.** There is no config load, no socket
/// server, no session, no brain and no diagnostics command — all of that went to
/// `Octavia.Server`, which can now be on another machine entirely. What is left is the part
/// that has to be on *this* one: a window, a tray, and a hotkey registered with this
/// Windows session.
public partial class App : Application
{
    private const string MutexName = @"Local\Octavia.SingleInstance";
    private const string SurfaceSignalName = @"Local\Octavia.Surface";

    private Mutex? _onlyOne;
    private EventWaitHandle? _surfaceSignal;
    private RegisteredWaitHandle? _surfaceWait;
    private NotifyIcon? _tray;
    private MainWindow? _window;
    private ClientConfig _client = new();
    /// True when startup bailed out before building anything, so OnExit tears down
    /// only what actually exists.
    private bool _exitedEarly;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        _onlyOne = new Mutex(true, MutexName, out var isFirst);
        if (!isFirst)
        {
            // She is already on screen, probably hidden in the tray. Bring that one forward
            // rather than exiting silently and looking broken.
            _exitedEarly = true;

            if (EventWaitHandle.TryOpenExisting(SurfaceSignalName, out var running))
            {
                running.Set();
                running.Dispose();
            }

            Shutdown();
            return;
        }

        _surfaceSignal = new EventWaitHandle(false, EventResetMode.AutoReset, SurfaceSignalName);
        _surfaceWait = ThreadPool.RegisterWaitForSingleObject(
            _surfaceSignal,
            (_, _) => Dispatcher.BeginInvoke(() => _window?.Surface()),
            null, Timeout.Infinite, false);

        WatchForCrashes();

        _client = ClientConfig.Load();

        // A shortcut can pass an argument but cannot set an environment variable, which is
        // why the command line is where a launcher says which server it wants.
        if (ValueArgument(e.Args, "--server", "-s") is { Length: > 0 } server) _client.Server = server;
        if (ValueArgument(e.Args, "--key") is { Length: > 0 } key) _client.Key = key;
        if (ValueArgument(e.Args, "--room", "-r") is { Length: > 0 } room) _client.Room = room;

        Log.Write($"Octavia client starting {SystemReport.Version}; her server is {_client.Origin}");

        // Before the window, because the window's first act is to load a page from it.
        var hosting = LocalServer.Ensure(_client, ValueArgument(e.Args, "--profile", "-p"));

        _window = new MainWindow(_client, _client.Hotkey, hosting);
        BuildTray();

        if (_client.StartMinimised) _window.Hide();
        else _window.Show();
    }

    /// Three ways this window can fail without anyone finding out. Swallowing the first one
    /// and carrying on — which is what it used to do — hides exactly the faults worth
    /// seeing, so each is now written down.
    ///
    /// Her *own* faults are the server's to catch and always were; this watches the shell.
    private void WatchForCrashes()
    {
        DispatcherUnhandledException += (_, args) =>
        {
            Log.Error("unhandled on the UI thread", args.Exception);
            // Still handled: a companion that dies on one bad frame helps nobody. The
            // difference is that it is now visible.
            args.Handled = true;
            _window?.ReportCrash(args.Exception);
        };

        AppDomain.CurrentDomain.UnhandledException += (_, args) =>
        {
            if (args.ExceptionObject is Exception ex) Log.Error("unhandled on a background thread", ex);
            else Log.Error($"unhandled on a background thread: {args.ExceptionObject}");
        };

        // A faulted task nobody awaited. Usually harmless, occasionally the only trace
        // of a subsystem that quietly stopped working.
        TaskScheduler.UnobservedTaskException += (_, args) =>
        {
            Log.Warn($"unobserved task exception: {args.Exception.Message}");
            args.SetObserved();
        };
    }

    /// --name value, --name=value, or an alias such as -s.
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

    private void BuildTray()
    {
        var menu = new ContextMenuStrip();
        menu.Items.Add("Show Octavia", null, (_, _) => _window?.Surface());
        menu.Items.Add($"Listen  ({_client.Hotkey})", null, (_, _) => _window?.ToggleListening());
        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add("Save diagnostics...", null, (_, _) => _window?.SaveDiagnostics());
        menu.Items.Add("Open data folder", null, (_, _) =>
            System.Diagnostics.Process.Start("explorer.exe", Paths.DataDir));
        /* **Starting and stopping her left this menu in v0.46.0**, and the reason is a rule
           rather than a problem with the entries: *"clients should not be able to configure
           server side things, they should only be able to set what they send out."*

           A client is a renderer with a microphone. Its business is what it sends her and what
           it draws; the lifetime of a service on somebody else's machine is not that, and the
           fact that this one usually runs on the same box is a coincidence of the moment
           rather than a licence. `Octavia.Control` owns it now — a tray of its own, in the
           user's session, where a service cannot draw one.

           `LocalServer.Ensure` stays. Attaching to a server, and starting one when there is
           none to attach to, is what makes double-clicking her work; it is not configuration,
           and removing it would leave the ordinary case broken to satisfy a tidy rule. */

        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add("Quit", null, (_, _) => Quit());

        _tray = new NotifyIcon
        {
            Icon = TrayIcon(),
            // Where she is, not what she is running. The profile and the brain belong to a
            // process this one cannot see until `hello` arrives, and a tooltip that guessed
            // would be wrong exactly when it mattered.
            Text = $"Octavia — {_client.Origin}",
            Visible = true,
            ContextMenuStrip = menu
        };

        _tray.DoubleClick += (_, _) => _window?.Surface();
    }

    /// Her mark, or the generic Windows box if the file is missing.
    ///
    /// A separate icon from the window's, and deliberately a rounder, simpler crop: the
    /// tray draws at 16 pixels — 20 or 24 on a high-DPI display — and the full app icon
    /// measured at that size is an unreadable smudge. `SystemIcons.Application` is still
    /// the fallback, because a missing asset should cost her the branding and nothing else.
    private static Icon TrayIcon()
    {
        var path = Path.Combine(AppContext.BaseDirectory, "Assets", "octavia-tray.ico");

        try
        {
            if (!File.Exists(path)) return SystemIcons.Application;

            // Asks for the 16px entry rather than letting GDI+ pick and rescale, which is
            // the difference between the artwork drawn for this size and a downsample of
            // the 48. SystemInformation gives the size the shell actually wants.
            var wanted = SystemInformation.SmallIconSize;
            return new Icon(path, wanted);
        }
        catch (Exception ex)
        {
            Log.Write($"tray icon could not be loaded: {ex.Message}");
            return SystemIcons.Application;
        }
    }

    private void Quit()
    {
        if (_window is not null)
        {
            _window.ShuttingDown = true;
            _window.Close();
        }

        // Her server is deliberately left running, whoever started it: a handset can be
        // mid-conversation while nobody is at this desk. See LocalServer for the three
        // mechanisms that were tried and why none of them belongs in a client.
        Shutdown();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        if (_exitedEarly)
        {
            _onlyOne?.Dispose();
            base.OnExit(e);
            return;
        }

        if (_tray is not null)
        {
            _tray.Visible = false;
            _tray.Dispose();
        }

        _surfaceWait?.Unregister(null);
        _surfaceSignal?.Dispose();
        _onlyOne?.Dispose();
        Log.Write("Octavia client stopped");
        base.OnExit(e);
    }
}
```
