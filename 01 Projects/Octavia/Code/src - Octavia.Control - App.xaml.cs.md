---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Control\App.xaml.cs
---

# src\Octavia.Control\App.xaml.cs

```csharp
using System.Drawing;
using System.IO;
using System.Windows;
using System.Windows.Forms;
using Octavia.Core;
using Application = System.Windows.Application;

namespace Octavia.Control;

/// The tray icon that controls her server.
///
/// **`ShutdownMode="OnExplicitShutdown"`** because there is no main window: this lives in the
/// tray with the settings window opened and closed as often as somebody likes, and the default
/// mode would quit the moment that window was closed for the first time.
public partial class App : Application
{
    private NotifyIcon? _tray;
    private SettingsWindow? _settings;

    /// Re-read every time the menu opens rather than cached. Somebody who has just started her
    /// from a shortcut, or stopped her to run a build, should not be shown what was true when
    /// this process launched.
    private ToolStripMenuItem _state = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // Only one, or two tray icons appear and both fight over the same service.
        _single = new Mutex(true, @"Local\Octavia.Control", out var mine);
        if (!mine)
        {
            System.Windows.MessageBox.Show(
                "Octavia's controls are already running — look in the notification area.",
                "Octavia", MessageBoxButton.OK, MessageBoxImage.Information);

            Shutdown();
            return;
        }

        BuildTray();

        if (e.Args.Contains("--settings", StringComparer.OrdinalIgnoreCase)) OpenSettings();
    }

    private Mutex? _single;

    private void BuildTray()
    {
        var menu = new ContextMenuStrip();

        _state = new ToolStripMenuItem("...") { Enabled = false };
        menu.Items.Add(_state);
        menu.Items.Add(new ToolStripSeparator());

        menu.Items.Add("Settings...", null, (_, _) => OpenSettings());
        menu.Items.Add(new ToolStripSeparator());

        menu.Items.Add("Start her", null, (_, _) => Then(() => ServerControl.Control(start: true)));
        menu.Items.Add("Stop her", null, (_, _) => Then(() => ServerControl.Control(start: false)));
        menu.Items.Add("Restart her", null, (_, _) => Then(ServerControl.Restart));

        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add("Open data folder", null, (_, _) =>
            System.Diagnostics.Process.Start("explorer.exe", Paths.DataDir));

        menu.Items.Add(new ToolStripSeparator());

        /* **Quits these controls, not her.** A tray icon whose Quit stopped a service that is
           meant to survive a reboot would be a trap, and the wording is the whole guard: the
           two are different actions and "Stop her" is three lines above. */
        menu.Items.Add("Quit these controls", null, (_, _) => Quit());

        // The state line is the reason the menu is worth opening at all, so it is refreshed
        // as it opens rather than on a timer nobody is watching.
        menu.Opening += (_, _) => _state.Text = Describe();

        _tray = new NotifyIcon
        {
            Icon = TrayIcon(),
            Text = "Octavia — her server",
            Visible = true,
            ContextMenuStrip = menu
        };

        _tray.DoubleClick += (_, _) => OpenSettings();
    }

    private static string Describe() => ServerControl.ServiceState switch
    {
        0 => "She is running",
        2 => "She is stopped",
        1 => "No service installed — run --install",
        _ => "Octavia.Server.exe was not found"
    };

    /// Runs a service command off the UI thread and reports the outcome in the balloon.
    ///
    /// Both can take seconds — her ears alone are 1.6 GB — so a menu that waited would freeze
    /// the desktop, and one that said nothing would leave somebody clicking it again.
    private void Then(Func<bool> action) => Task.Run(() =>
    {
        var worked = action();

        Dispatcher.Invoke(() => _tray?.ShowBalloonTip(
            3000, "Octavia", worked ? Describe() : "That did not work — her log says why.",
            worked ? ToolTipIcon.Info : ToolTipIcon.Warning));
    });

    private void OpenSettings()
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

    private void Quit()
    {
        if (_tray is not null) { _tray.Visible = false; _tray.Dispose(); }
        _single?.Dispose();
        Shutdown();
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
}
```
