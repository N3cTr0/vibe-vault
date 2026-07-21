---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\MainWindow.xaml.cs
---

# PartnerTool\MainWindow.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media;
using System.Windows.Threading;
using PartnerTool.Pages;

namespace PartnerTool;

public partial class MainWindow : Window
{
    private Button? _activeNav;

    private readonly SystemInfoPage _sysInfoPage  = new();
    private readonly NetworkPage    _networkPage  = new();
    private readonly SecurityPage   _securityPage = new();
    private readonly DiagnosticsPage _diagPage    = new();
    private readonly ManagePage     _managePage   = new();
    private readonly ShortcutsPage  _shortcutsPage = new();
    private readonly SoftwarePage   _softwarePage = new();
    private readonly UpdatesPage    _updatesPage  = new();
    private readonly RepairPage     _repairPage   = new();
    private readonly DiskUsagePage  _diskUsagePage = new();
    private readonly HealthCheckPage _healthPage  = new();

    public MainWindow()
    {
        InitializeComponent();
        InitGateIndicator();
        Loaded += async (_, _) => await StartupAsync();
    }

    // ── Tech-gate lock indicator (header) ────────────────────────────────────────────────
    // Padlock + countdown so a tech can see at a glance whether this machine is still unlocked,
    // and lock it back down by hand before walking away instead of waiting out the idle timeout.

    private const string LockGlyph   = "\uE72E";   // Segoe MDL2 Assets "Lock"   - closed padlock
    private const string UnlockGlyph = "\uE785";   // Segoe MDL2 Assets "Unlock" - open padlock

    private static readonly Brush GateAmberBg  = Frozen(0x3A, 0x2E, 0x1E);
    private static readonly Brush GateAmberFg  = Frozen(0xF9, 0xE2, 0xAF);
    private static readonly Brush GateLockedFg = Frozen(0x6C, 0x70, 0x86);

    private static SolidColorBrush Frozen(byte r, byte g, byte b)
    {
        var brush = new SolidColorBrush(Color.FromRgb(r, g, b));
        brush.Freeze();
        return brush;
    }

    private readonly DispatcherTimer _gateTimer = new() { Interval = TimeSpan.FromSeconds(1) };
    private bool _gateWasUnlocked;

    private void InitGateIndicator()
    {
        _gateTimer.Tick += (_, _) => PaintGate();
        _gateTimer.Start();
        TechGate.Changed += PaintGate;   // repaint the instant the gate flips, not up to a second later
        PaintGate();
    }

    /// <summary>
    /// Repaints the header padlock. This runs once a second to drive the countdown, so it only
    /// touches the chrome (glyph, brushes, tooltip) when the state actually flips - a locked app,
    /// which is the common case, does no work at all.
    /// </summary>
    private void PaintGate()
    {
        bool unlocked = TechGate.IsUnlocked;
        bool flipped  = unlocked != _gateWasUnlocked;
        if (!unlocked && !flipped) return;   // locked and already showing it
        _gateWasUnlocked = unlocked;

        if (unlocked)
        {
            var left = TechGate.Remaining;
            TxtGateText.Text = $"{(int)left.TotalMinutes}:{left.Seconds:00}";
            BtnGate.ToolTip  = $"Tech gate UNLOCKED - {(int)left.TotalMinutes}m {left.Seconds}s left.\n" +
                               "Destructive actions won't ask for the code until it expires.\nClick to lock now.";
            if (!flipped) return;            // countdown tick only; chrome is already amber/open

            TxtGateIcon.Text   = UnlockGlyph;
            BtnGate.Background = GateAmberBg;
            TxtGateIcon.Foreground = TxtGateText.Foreground = GateAmberFg;
        }
        else
        {
            TxtGateIcon.Text   = LockGlyph;
            TxtGateText.Text   = "Locked";
            BtnGate.Background = Brushes.Transparent;
            TxtGateIcon.Foreground = TxtGateText.Foreground = GateLockedFg;
            BtnGate.ToolTip = "Tech gate LOCKED - the next destructive action will ask for the tech code.\n" +
                              "Click to unlock now.";
        }
    }

    /// <summary>Locked → offer to unlock (still needs the code). Unlocked → lock out immediately.</summary>
    private void Gate_Click(object sender, RoutedEventArgs e)
    {
        if (TechGate.IsUnlocked) TechGate.Lock();
        else TechGate.Verify(this);
        PaintGate();
    }

    /// <summary>
    /// Behind the loading splash: capture one system snapshot, paint the heavy
    /// info pages from it (so they're instant from then on), land on System Info.
    /// </summary>
    private async Task StartupAsync()
    {
        try
        {
            var warmUp = _sysInfoPage.WarmUpLiveAsync();   // prime the live tiles behind the splash
            var snap = await SystemSnapshot.CaptureAsync();
            SnapshotHistory.Append(snap);   // record today's headline metrics for the trend view
            _sysInfoPage.Render(snap);
            _diagPage.Render(snap);
            await warmUp;   // live tiles are primed
            // Warm the slow pages in the background while the tech reads System Info, so they're
            // already populated when opened: all Manage sections, the installed-software +
            // winget-outdated scans, and the update-source scans.
            WingetLocator.Warm();   // resolve winget.exe off-thread so no winget action stalls on it
            _ = _managePage.PreloadAsync();
            _ = _softwarePage.EnsureLoadedAsync();
            _ = _updatesPage.EnsureLoadedAsync();
        }
        catch { /* every collector already fails safe; never let the splash hang */ }
        finally
        {
            Activate(BtnNavSysInfo, _sysInfoPage);
            LoadingOverlay.Visibility = Visibility.Collapsed;
            // In a hidden session (Backstage) confirm we got all the way to a visible UI, so a
            // future "doesn't load" report tells us whether startup or rendering is the problem.
            if (App.SoftwareRendering)
                App.Breadcrumb("main window rendered, splash dismissed (software rendering active)");
        }
    }

    protected override void OnSourceInitialized(EventArgs e)
    {
        base.OnSourceInitialized(e);

        // Keep the window inside the screen's working area so the title bar — and its
        // close button — is always reachable. A fixed 900px height overflowed the top
        // of the screen on laptops at lower resolutions or display scaling: the window
        // was centred, pushing the title bar above the top edge, and CanMinimize left
        // no way to recover. WorkArea is in the same device-independent units as Width/Height.
        var wa = SystemParameters.WorkArea;
        if (Width  > wa.Width)  Width  = wa.Width;
        if (Height > wa.Height) Height = wa.Height;
        Left = wa.Left + Math.Max(0, (wa.Width  - Width)  / 2);
        Top  = wa.Top  + Math.Max(0, (wa.Height - Height) / 2);
    }

    private void NavBtn_Click(object sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is string tag) NavigateTo(tag);
    }

    internal void NavigateTo(string tag)
    {
        Button btn = tag switch
        {
            "Network"   => BtnNavNetwork,
            "Security"  => BtnNavSecurity,
            "Diag"      => BtnNavDiag,
            "Manage"    => BtnNavManage,
            "Shortcuts" => BtnNavShortcuts,
            "Software"  => BtnNavSoftware,
            "Updates"   => BtnNavUpdates,
            "Repair"    => BtnNavRepair,
            "DiskUsage" => BtnNavDiskUsage,
            "Health"    => BtnNavHealth,
            _           => BtnNavSysInfo,
        };
        UserControl page = tag switch
        {
            "Network"   => _networkPage,
            "Security"  => _securityPage,
            "Diag"      => _diagPage,
            "Manage"    => _managePage,
            "Shortcuts" => _shortcutsPage,
            "Software"  => _softwarePage,
            "Updates"   => _updatesPage,
            "Repair"    => _repairPage,
            "DiskUsage" => _diskUsagePage,
            "Health"    => _healthPage,
            _           => _sysInfoPage,
        };
        Activate(btn, page);
    }

    private void Activate(Button btn, UserControl page)
    {
        if (_activeNav != null)
            _activeNav.Style = (Style)FindResource("NavBtn");
        btn.Style = (Style)FindResource("NavBtnActive");
        _activeNav = btn;
        PageHost.Content = page;
    }

    private void About_Click(object sender, RoutedEventArgs e)
        => new AboutWindow { Owner = this }.ShowDialog();

    private void Settings_Click(object sender, RoutedEventArgs e)
        => new SettingsWindow { Owner = this }.ShowDialog();

    [System.Runtime.InteropServices.DllImport("kernel32.dll")]
    private static extern IntPtr GetCurrentProcess();
    [System.Runtime.InteropServices.DllImport("kernel32.dll")]
    private static extern bool TerminateProcess(IntPtr hProcess, uint exitCode);

    protected override void OnClosed(EventArgs e)
    {
        base.OnClosed(e);
        // Hard-terminate the process rather than letting the runtime shut down. Two separate
        // exit-time faults were logged as WER "Application Error" crashes on close:
        //   1) WPF shutdown telemetry (ControlsTraceLogger) name-loads System.Diagnostics.Tracing,
        //      which fails in a single-file publish → FileNotFoundException.
        //   2) Environment.Exit still runs the native C++ CRT module-unload callbacks
        //      (_app_exit_callback / __scrt_uninitialize_type_info); a native lib extracted to the
        //      single-file temp dir can already be gone by then → DllNotFoundException.
        // Nothing needs graceful shutdown — every log and setting is written synchronously as it
        // changes — so TerminateProcess ends the process immediately, skipping ALL teardown
        // (managed finalizers, native dtors, atexit). No exit-time fault is possible.
        TerminateProcess(GetCurrentProcess(), 0);
    }
}
```
