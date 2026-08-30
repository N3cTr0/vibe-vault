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

public partial class App : Application
{
    private const string MutexName = @"Local\Octavia.SingleInstance";
    private const string SurfaceSignalName = @"Local\Octavia.Surface";

    private Mutex? _onlyOne;
    private EventWaitHandle? _surfaceSignal;
    private RegisteredWaitHandle? _surfaceWait;
    private NotifyIcon? _tray;
    private MainWindow? _window;
    private OctaviaConfig _config = new();
    /// True when startup bailed out before building anything, so OnExit tears down
    /// only what actually exists.
    private bool _exitedEarly;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var requestedProfile = ValueArgument(e.Args, "--profile", "-p");

        // Before the single-instance check on purpose: this has to work when she is
        // already running, and — more to the point — when she is too broken to start.
        var bundlePath = ValueArgument(e.Args, "--diagnostics");
        if (bundlePath is not null)
        {
            WriteBundleAndExit(bundlePath, requestedProfile);
            return;
        }

        _onlyOne = new Mutex(true, MutexName, out var isFirst);
        if (!isFirst)
        {
            // She is already running, probably hidden in the tray. Bring that one
            // forward rather than exiting silently and looking broken.
            _exitedEarly = true;
            if (requestedProfile is not null)
            {
                Log.Write($"already running, so profile '{requestedProfile}' was ignored; " +
                          "quit her from the tray first to switch profiles");
            }

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

        _config = OctaviaConfig.Load(requestedProfile);
        Log.Write($"Octavia starting {SystemReport.Version}; data in {Paths.DataDir}");

        _window = new MainWindow(_config);
        BuildTray();

        if (_config.StartMinimised) _window.Hide();
        else _window.Show();
    }

    /// Three ways she can fail without anyone finding out. Swallowing the first one and
    /// carrying on — which is what she used to do — hides exactly the faults worth
    /// seeing, so each is now written down, and the UI thread's is shown to the person
    /// in front of her.
    private void WatchForCrashes()
    {
        DispatcherUnhandledException += (_, args) =>
        {
            Log.Error("unhandled on the UI thread", args.Exception);
            // Still handled: she is a companion, not a batch job, and dying on one bad
            // frame helps nobody. The difference is that it is now visible.
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

    /// Writes a bundle with no window and no session, then leaves. The one thing a
    /// person can still do when she will not start at all — the environment, the
    /// settings and the logs are exactly what explains why.
    private void WriteBundleAndExit(string path, string? requestedProfile)
    {
        _exitedEarly = true;

        try
        {
            _config = OctaviaConfig.Load(requestedProfile);
            // On the thread pool, not here: blocking the dispatcher thread on a task
            // whose continuations are posted back to it is a deadlock, and a headless
            // command that hangs forever is worse than one that fails.
            Task.Run(() => DiagnosticsBundle.WriteAsync(path, _config, HostSnapshot.Stopped))
                .GetAwaiter().GetResult();
            Console.WriteLine($"Diagnostics written to {path}");
        }
        catch (Exception ex)
        {
            Log.Error("headless diagnostics failed", ex);
            Console.Error.WriteLine($"Could not write diagnostics: {ex.Message}");
        }

        Shutdown();
    }

    /// --name value, --name=value, or an alias such as -p. A desktop shortcut can pass
    /// an argument but cannot set an environment variable, which is why the command
    /// line is where a launcher states what it wants.
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
        menu.Items.Add($"Listen  ({_config.Hotkey})", null, (_, _) => _window?.ToggleListening());
        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add("Save diagnostics...", null, (_, _) => _window?.SaveDiagnostics());
        menu.Items.Add("Open data folder", null, (_, _) =>
            System.Diagnostics.Process.Start("explorer.exe", Paths.DataDir));
        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add("Quit", null, (_, _) => Quit());

        _tray = new NotifyIcon
        {
            Icon = SystemIcons.Application,
            Text = $"Octavia — {_config.Profile} ({_config.Brain})",
            Visible = true,
            ContextMenuStrip = menu
        };

        _tray.DoubleClick += (_, _) => _window?.Surface();
    }

    private void Quit()
    {
        if (_window is not null)
        {
            _window.ShuttingDown = true;
            _window.Close();
        }

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
        Log.Write("Octavia stopped");
        base.OnExit(e);
    }
}
```
