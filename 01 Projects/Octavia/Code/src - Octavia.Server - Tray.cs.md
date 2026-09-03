---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Tray.cs
---

# src\Octavia.Server\Tray.cs

```csharp
using System.Drawing;
using System.Windows.Forms;
using Octavia.Core;
using Application = System.Windows.Application;

namespace Octavia.Server;

/// The tray icon, and the whole of managing her from a desktop.
///
/// **This is the same executable as the service**, and that is the point of v0.47.0. A
/// Windows service has no desktop, so an icon drawn from session 0 would be drawn where
/// nobody can see it — which is true, and which never required a second *binary*. It
/// required a second *mode*. The tray is only ever launched by a person in their own
/// session; the service never draws anything.
///
/// **It manages the service rather than hosting her.** A tray that also ran her in-process
/// would have two ways to be running, two things for the settings window to restart, and a
/// race with the service over the port. `--console` is the answer for running her without a
/// service, and it is one line in the menu.
internal sealed class Tray : IDisposable
{
    private const string MutexName = @"Local\Octavia.Tray";

    private readonly string? _profile;
    private NotifyIcon? _icon;
    private SettingsWindow? _settings;
    private Mutex? _single;

    private Tray(string? profile) => _profile = profile;

    /// Shows the tray and runs a message loop until Quit. Returns the process exit code.
    public static int Run(string? profile, bool settings = false)
    {
        using var tray = new Tray(profile);

        tray._single = new Mutex(true, MutexName, out var mine);

        if (!mine)
        {
            /* Two icons would both control one service, and each would show the other's
               changes as though they had happened by themselves.

               **Printed rather than shown in a dialog**, which is what this did first: a
               modal box means the refused process *stays alive* holding it, so somebody who
               double-clicks a shortcut three times collects three windows and three
               processes to dismiss. The console is still attached at this point — `FreeConsole`
               is below — so a person who typed the command sees this, and a person who
               double-clicked sees the icon that is already there. */
            Console.Error.WriteLine("Octavia's tray is already running — look in the notification area.");
            return 1;
        }

        /* The console goes now, and not before: everything above this line may still need to
           print. It closes the window a double-click opened, and merely detaches from a
           terminal's — see `Program.FreeConsole` for why she is not a `WinExe`. */
        Program.FreeConsole();

        var application = new Application { ShutdownMode = System.Windows.ShutdownMode.OnExplicitShutdown };
        tray.Build();

        if (settings) tray.OpenSettings();

        return application.Run();
    }

    private void Build()
    {
        var menu = new ContextMenuStrip();

        var state = new ToolStripMenuItem("...") { Enabled = false };
        menu.Items.Add(state);
        menu.Items.Add(new ToolStripSeparator());

        menu.Items.Add("Settings...", null, (_, _) => OpenSettings());
        menu.Items.Add(new ToolStripSeparator());

        var install = new ToolStripMenuItem("Install her as a service...", null,
            (_, _) => Then(() => ServerControl.Install(_profile)));

        var start = new ToolStripMenuItem("Start her", null, (_, _) => Then(() => ServerControl.Control(start: true)));
        var stop = new ToolStripMenuItem("Stop her", null, (_, _) => Then(() => ServerControl.Control(start: false)));
        var restart = new ToolStripMenuItem("Restart her", null, (_, _) => Then(ServerControl.Restart));

        menu.Items.AddRange([install, start, stop, restart]);
        menu.Items.Add(new ToolStripSeparator());

        menu.Items.Add("Run her in a window instead", null, (_, _) => RunConsole());
        menu.Items.Add("Open data folder", null, (_, _) =>
            System.Diagnostics.Process.Start("explorer.exe", Paths.DataDir));

        menu.Items.Add(new ToolStripSeparator());

        /* **Quits the tray, not her.** A tray whose Quit stopped a service that is meant to
           survive a reboot would be a trap, and the wording is the whole guard — "Stop her"
           is four lines above it. */
        menu.Items.Add("Quit this tray", null, (_, _) => Quit());

        /* Read as the menu opens rather than on a timer.

           Somebody who has just started her from a shortcut, stopped her to run a build, or
           installed the service in another window should not be shown what was true when
           this process launched — and a menu nobody has opened is not worth polling for. */
        menu.Opening += (_, _) =>
        {
            var installed = ServerControl.ServiceState != 1;

            state.Text = Describe();
            install.Visible = !installed;
            start.Visible = stop.Visible = restart.Visible = installed;
        };

        _icon = new NotifyIcon
        {
            Icon = TrayIcon(),
            Text = "Octavia",
            Visible = true,
            ContextMenuStrip = menu
        };

        _icon.DoubleClick += (_, _) => OpenSettings();

        /* Says where she went, once, on the launch that put her there.

           Without it, double-clicking her shortcut appears to do nothing at all — the window
           people expect never opens, because there is no window. This is the other half of
           refusing a second instance quietly: the first launch has to be discoverable, or the
           second one is somebody trying again. */
        _icon.ShowBalloonTip(4000, "Octavia", $"{Describe()}. Right-click here to manage her.", ToolTipIcon.None);
    }

    private static string Describe() => ServerControl.ServiceState switch
    {
        0 => "She is running",
        2 => "She is stopped",
        1 => "Not installed as a service yet",
        _ => "Octavia.Server.exe was not found"
    };

    /// Runs a service command off the UI thread and says what happened in the balloon.
    ///
    /// Both can take seconds — her ears alone are 1.6 GB — so a menu that waited would freeze
    /// the desktop, and one that said nothing would leave somebody clicking it again.
    private void Then(Func<bool> action) => Task.Run(() =>
    {
        var worked = action();

        _icon?.ShowBalloonTip(3000, "Octavia",
            worked ? Describe() : "That did not work — her log says why.",
            worked ? ToolTipIcon.Info : ToolTipIcon.Warning);
    });

    /// Her, in a console window, with no service involved. The same executable again, with
    /// `--console`: it is how she was run before there was a service, and it is still the
    /// right thing when somebody wants to watch her start.
    private void RunConsole()
    {
        try
        {
            var arguments = _profile is { Length: > 0 } ? $"--console --profile {_profile}" : "--console";

            System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo(
                Environment.ProcessPath ?? "Octavia.Server.exe", arguments) { UseShellExecute = true });
        }
        catch (Exception ex)
        {
            Log.Warn($"could not start her in a window: {ex.Message}");
        }
    }

    /// Opens the settings window, and **survives it failing to open**.
    ///
    /// Without the catch this took the whole tray down, silently: the console has been freed
    /// by the time anything here runs, so an exception out of `Main` closes the process with
    /// nowhere to print. A tray that vanishes when you ask it for settings is indistinguishable
    /// from one that was never running.
    private void OpenSettings()
    {
        try
        {
            if (_settings is { IsLoaded: true })
            {
                _settings.Activate();
                return;
            }

            _settings = new SettingsWindow();
            _settings.Show();
            _settings.Activate();
        }
        catch (Exception ex)
        {
            Log.Error("her settings window would not open", ex);

            _icon?.ShowBalloonTip(5000, "Octavia",
                "Her settings would not open — her log says why.", ToolTipIcon.Error);
        }
    }

    private void Quit()
    {
        Dispose();
        Application.Current?.Shutdown();
    }

    /// Her mark, or the generic Windows box if the file is missing — a missing asset should
    /// cost the branding and nothing else.
    private static Icon TrayIcon()
    {
        var path = Path.Combine(AppContext.BaseDirectory, "Assets", "octavia-tray.ico");

        try { return File.Exists(path) ? new Icon(path) : SystemIcons.Application; }
        catch (Exception ex)
        {
            Log.Warn($"could not load the tray icon: {ex.Message}");
            return SystemIcons.Application;
        }
    }

    public void Dispose()
    {
        if (_icon is not null) { _icon.Visible = false; _icon.Dispose(); _icon = null; }
        _single?.Dispose();
        _single = null;
    }
}
```
