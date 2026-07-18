---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SystemInfoPage.xaml.cs
---

# PartnerTool\Pages\SystemInfoPage.xaml.cs

```csharp
using System.Diagnostics;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media;
using System.Windows.Shapes;
using System.Windows.Threading;

namespace PartnerTool.Pages;

public partial class SystemInfoPage : UserControl
{
    // Kept from the last paint so the Warranty lookup has the maker + serial to query.
    private string _manufacturer = "";
    private string _serial       = "";

    // Live performance widget (top-left): sampled off the UI thread on a timer, paused off-screen.
    private const int Cap = 60;   // points kept per sparkline
    private readonly DispatcherTimer _liveTimer = new() { Interval = TimeSpan.FromSeconds(1) };
    // Auto-refresh the (non-live) snapshot data every 45s. Uses Normal priority and runs for the
    // life of the page (work is gated on IsVisible inside the tick) so it can't be left un-started
    // by a missed visibility change — the manual Refresh button is gone, so this must be reliable.
    private const int RefreshSeconds = 45;
    private readonly DispatcherTimer _autoRefresh =
        new(DispatcherPriority.Normal) { Interval = TimeSpan.FromSeconds(RefreshSeconds) };
    private bool _loading;      // guards against overlapping refreshes (auto + manual)
    private readonly LivePerf _live = new();
    private readonly List<double> _cpu = new(), _ram = new(), _disk = new(), _net = new();
    private bool _liveTicking;
    private bool _hasBattery;   // set from the snapshot; gates the live battery poll to laptops
    private int  _tick;         // battery is polled every 3rd tick — its rate needn't be 1 Hz

    public SystemInfoPage()
    {
        InitializeComponent();
        _liveTimer.Tick   += async (_, _) => await LiveTick();
        // Always-on timer; only does work while the page is visible. Started here (not in
        // IsVisibleChanged) so it's guaranteed to run.
        _autoRefresh.Tick += async (_, _) => { if (IsVisible) await LoadAsync(); };
        _autoRefresh.Start();
        IsVisibleChanged += (_, _) =>
        {
            if (IsVisible) { _liveTimer.Start(); _ = LiveTick(); }
            else _liveTimer.Stop();
        };
    }

    private async Task LiveTick()
    {
        if (_liveTicking) return;       // skip if the previous sample is still running
        _liveTicking = true;
        try
        {
            // WMI disk / NIC reads can take 50–200ms — keep them off the UI thread. Do the perf
            // sample (and, every 3rd tick, the battery sample) in a single thread-pool hop.
            bool pollBattery = _hasBattery && _tick % 3 == 0;
            var (s, b) = await Task.Run(() =>
                (_live.Sample(), pollBattery ? BatteryLive.Sample() : null));
            _tick++;
            if (!IsVisible) return;
            ApplyLiveSample(s);
            if (b is { Present: true })
            {
                TxtBattPower.Text       = b.Summary;
                TxtBattPower.Foreground = b.OnAc ? StatusColors.Green : StatusColors.Yellow;
                TxtBattVoltage.Text     = b.VoltageV is { } vv ? $"{vv:F1} V" : "—";
            }
        }
        catch { }
        finally { _liveTicking = false; }
    }

    /// <summary>
    /// Prime the live widget behind the startup splash. CPU% and network throughput are
    /// computed from the delta against the previous reading, so the very first sample reads 0;
    /// taking a few here means the tiles already show real numbers (and a little graph history)
    /// the instant System Info is shown, instead of looking stuck for a second or two.
    /// </summary>
    public async Task WarmUpLiveAsync()
    {
        try
        {
            for (int i = 0; i < 3; i++)
            {
                var s = await Task.Run(() => _live.Sample());
                ApplyLiveSample(s);     // page isn't visible yet — populate it anyway
                await Task.Delay(250);
            }
        }
        catch { }
    }

    private void ApplyLiveSample(LiveSample s)
    {
        TxtLiveCpu.Text  = $"{s.CpuPct:F0}%";
        TxtLiveRam.Text  = $"{s.RamPct:F0}%";
        TxtLiveDisk.Text = $"{s.DiskPct:F0}%";
        TxtLiveNet.Text  = $"↓{s.NetDownMbps:F1}  ↑{s.NetUpMbps:F1}";

        Push(_cpu, s.CpuPct); Push(_ram, s.RamPct); Push(_disk, s.DiskPct);
        Push(_net, s.NetDownMbps + s.NetUpMbps);
        Plot(LiveCpuPlot,  LiveCpuLine,  _cpu,  100);
        Plot(LiveRamPlot,  LiveRamLine,  _ram,  100);
        Plot(LiveDiskPlot, LiveDiskLine, _disk, 100);
        Plot(LiveNetPlot,  LiveNetLine,  _net,  Math.Max(10, _net.Count > 0 ? _net.Max() : 10));
    }

    private static void Push(List<double> data, double v)
    {
        data.Add(v);
        if (data.Count > Cap) data.RemoveAt(0);
    }

    private static void Plot(Border host, Polyline line, List<double> data, double max)
    {
        double w = host.ActualWidth, h = host.ActualHeight;
        if (w <= 0 || h <= 0 || data.Count < 2 || max <= 0) { line.Points = null; return; }
        var pts = new PointCollection();
        for (int i = 0; i < data.Count; i++)
        {
            double x = w * i / (Cap - 1);
            double y = h - Math.Clamp(data[i] / max, 0, 1) * h;
            pts.Add(new Point(x, y));
        }
        line.Points = pts;
    }

    // Clicking a LIVE STATS tile opens the Task-Manager-style Performance window.
    private PerformanceWindow? _perfWin;
    private void LiveTile_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not FrameworkElement { Tag: string res }) return;
        if (_perfWin != null) { _perfWin.SelectResource(res); _perfWin.Activate(); return; }
        _perfWin = new PerformanceWindow(res) { Owner = Window.GetWindow(this) };
        _perfWin.Closed += (_, _) => _perfWin = null;
        _perfWin.Show();
    }

    /// <summary>Paint immediately from the startup snapshot (no collection).</summary>
    public void Render(SystemSnapshot snap)
        => Paint(snap.Info, snap.Perf, snap.Security, snap.PrimaryAdapter, snap.Hardware,
                 snap.Temps, snap.Displays, snap.Printers, snap.Accounts, snap.Extras, snap.Aad, snap.Power, snap.CapturedAt);

    /// <summary>Collapse the device-join state into a single Domain value: Azure AD joined /
    /// the domain (hybrid if also Azure AD) / else the workgroup name.</summary>
    private static string DomainText(SystemInfo info, AzureAdInfo aad)
    {
        if (aad.AzureAdJoined && !aad.DomainJoined)
            return string.IsNullOrWhiteSpace(aad.TenantName) || aad.TenantName == "—"
                ? "Azure AD Joined"
                : $"Azure AD Joined · {aad.TenantName}";
        if (aad.DomainJoined)
            return aad.EnterpriseJoined ? $"{info.Domain}  (hybrid Azure AD)" : info.Domain;
        return info.Domain;   // WORKGROUP / workgroup name
    }

    // ── Warranty lookup (moved here from the old Overview tile) ───────────
    private void Warranty_Click(object sender, RoutedEventArgs e)
    {
        try
        {
            // ManufacturerModel is "Maker, Model" — the maker is what the lookup needs.
            var maker = _manufacturer.Split(',')[0].Trim();
            var url   = WarrantyLink.For(maker, _serial);
            bool prefilled = WarrantyLink.IsPrefilled(maker);

            if (!prefilled) { try { Clipboard.SetText(_serial); } catch { } }

            Process.Start(new ProcessStartInfo(url) { UseShellExecute = true });

            if (!prefilled)
                MessageWindow.Show("Warranty Lookup", "Serial copied to clipboard",
                    $"Opened {maker}'s warranty page.\n\nThe service tag / serial ({_serial}) has been " +
                    "copied — paste it into the lookup form.", MessageKind.Info, Window.GetWindow(this));
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Warranty Lookup", "Couldn't open the warranty page",
                ex.Message, MessageKind.Error, Window.GetWindow(this));
        }
    }

    private void ChangePowerPlan_Click(object sender, RoutedEventArgs e)
    {
        new PowerPlanWindow { Owner = Window.GetWindow(this) }.ShowDialog();
        // Reflect whatever's now active without a full page refresh.
        try { TxtPwrMode.Text = PowerMode.ActiveName(); ActivityLog.Action("Power", $"Power mode now: {PowerMode.ActiveName()}"); } catch { }
    }

    // Fast startup + hibernation both live on the classic "Choose what the power buttons do" page
    // (pageGlobalSettings). control.exe is pinned to System32 so the bare name can't be hijacked.
    private void ChangeStartupSettings_Click(object sender, RoutedEventArgs e)
    {
        ActivityLog.Action("Power", "Open power-button/fast-startup settings");
        try
        {
            Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemExe("control.exe"),
                "/name Microsoft.PowerOptions /page pageGlobalSettings") { UseShellExecute = false });
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Power Settings", "Couldn't open the settings page", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }

    // Toggle hibernation directly — the settings page has no obvious switch for it, so the link
    // that just opened Power Options left techs hunting. powercfg /hibernate needs admin (we are).
    // Side effect worth knowing: OFF also deletes hiberfil.sys and disables fast startup (hiberboot
    // needs the hibernation file); the fast-startup row refreshes to reflect that.
    private async void HibToggle_Click(object sender, RoutedEventArgs e)
    {
        bool enable = !TxtPwrHib.Text.StartsWith("Enabled", StringComparison.OrdinalIgnoreCase);
        LnkHibToggle.Text = "(working…)";
        try
        {
            await ProcessRunner.RunAsync("powercfg.exe", enable ? "/hibernate on" : "/hibernate off",
                null, _ => { });
            var p = await PowerStatusInfo.CollectAsync();   // re-read: hib + fast startup both change
            TxtPwrHib.Text  = p.Hibernation;
            TxtPwrFast.Text = p.FastStartup;
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Hibernation", "Couldn't change hibernation", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
        finally { UpdateHibToggleText(); }
    }

    private void UpdateHibToggleText()
        => LnkHibToggle.Text = TxtPwrHib.Text.StartsWith("Enabled", StringComparison.OrdinalIgnoreCase)
            ? "(disable)" : "(enable)";

    // Sleep / display-off timeouts: the modern Settings ▸ System ▸ Power & sleep page.
    private void ChangeSleepSettings_Click(object sender, RoutedEventArgs e)
    {
        ActivityLog.Action("Power", "Open Power & sleep settings");
        try { Process.Start(new ProcessStartInfo("ms-settings:powersleep") { UseShellExecute = true }); }
        catch (Exception ex)
        {
            MessageWindow.Show("Power Settings", "Couldn't open Power & sleep settings", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }

    // Full CrystalDiskInfo-style SMART attribute table (SATA/ATA; empty on NVMe, which the window
    // explains). Collected off the UI thread — the WMI reads can take a moment.
    private async void Smart_Click(object sender, RoutedEventArgs e)
    {
        BtnSmart.IsEnabled = false;
        try
        {
            var drives = await Task.Run(SmartAttributes.Collect);
            new SmartWindow(drives) { Owner = Window.GetWindow(this) }.ShowDialog();
        }
        catch (Exception ex)
        {
            MessageWindow.Show("SMART", "Couldn't read SMART data", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
        finally { BtnSmart.IsEnabled = true; }
    }

    // ── POWER (moved from the old Actions tab) ────────────────────────────
    private bool Confirm(string msg, string title)
        => TechGate.Verify(Window.GetWindow(this))
           && MessageWindow.Confirm(title, title, msg, MessageKind.Warning, Window.GetWindow(this));

    private static void Spawn(string exe, string args)
    {
        ActivityLog.Command("Power", exe, args);   // restart / shutdown / sign-out / lock
        Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemExe(exe), args) { UseShellExecute = false, CreateNoWindow = true });
    }

    private void Restart_Click(object sender, RoutedEventArgs e)
    {
        if (Confirm("Restart this computer now?", "Restart")) Spawn("shutdown.exe", "/r /t 0");
    }

    private void Shutdown_Click(object sender, RoutedEventArgs e)
    {
        if (Confirm("Shut down this computer now?", "Shut Down")) Spawn("shutdown.exe", "/s /t 0");
    }

    private void Lock_Click(object sender, RoutedEventArgs e)
        => Spawn("rundll32.exe", "user32.dll,LockWorkStation");

    private void SignOut_Click(object sender, RoutedEventArgs e)
    {
        if (Confirm("Sign out the current user? Unsaved work will be lost.", "Sign Out"))
            Spawn("shutdown.exe", "/l");
    }

    private async Task LoadAsync()
    {
        if (_loading) return;   // don't let two refreshes overlap
        _loading = true;
        try
        {
            TxtRefreshed.Text = "Refreshing…";

            var infoTask  = Task.Run(SystemInfo.Collect);
            var perfTask  = Task.Run(PerfSnapshot.Collect);
            var secTask   = Task.Run(SecuritySnapshot.Collect);
            var netTask   = Task.Run(NetworkInfo.GetPrimaryAdapter);
            var hwTask    = Task.Run(HardwareInfo.Collect);
            var tempTask  = Task.Run(TemperatureInfo.Collect);
            var dispTask  = Task.Run(DisplaysInfo.Collect);
            var printTask = Task.Run(PrintersInfo.Collect);
            var acctTask  = Task.Run(AccountsInfo.Collect);
            var extraTask = Task.Run(SystemExtras.Collect);
            var aadTask   = AzureAdInfo.CollectAsync();
            var pwrTask   = PowerStatusInfo.CollectAsync();
            await Task.WhenAll(infoTask, perfTask, secTask, netTask, hwTask,
                               tempTask, dispTask, printTask, acctTask, extraTask, aadTask, pwrTask);

            Paint(await infoTask, await perfTask, await secTask, await netTask, await hwTask,
                  await tempTask, await dispTask, await printTask, await acctTask, await extraTask, await aadTask, await pwrTask, DateTime.Now);
        }
        catch (Exception ex)
        {
            // A single collector fault must never break the auto-refresh loop or crash the app.
            TxtRefreshed.Text = $"Last refresh failed at {DateTime.Now:HH:mm:ss} — {ex.Message}";
        }
        finally { _loading = false; }
    }

    private void Paint(SystemInfo info, PerfSnapshot perf, SecuritySnapshot sec,
                       AdapterSummary? adapter, HardwareInfo hw,
                       TemperatureInfo temps, List<DisplayInfo> displays, List<PrinterInfo> printers,
                       List<LocalAccount> accounts, SystemExtras extras, AzureAdInfo aad,
                       PowerStatusInfo power, DateTime asOf)
    {
        // ── Identity ──────────────────────────────────────────
        TxtHostname.Text = info.Hostname;
        TxtUser.Text     = info.LoggedInUser;
        TxtDomain.Text   = DomainText(info, aad);
        TxtTimezone.Text = info.Timezone;

        // ── Hardware ──────────────────────────────────────────
        TxtModel.Text  = info.ManufacturerModel;
        TxtSerial.Text = info.SerialNumber;
        _manufacturer  = info.ManufacturerModel;
        _serial        = info.SerialNumber;
        var warrantyVis =
            !string.IsNullOrWhiteSpace(_serial) && _serial != "Unknown"
                ? Visibility.Visible : Visibility.Collapsed;
        WarrantyLabel.Visibility   = warrantyVis;
        WarrantyLine.Visibility    = warrantyVis;
        WarrantyDivider.Visibility = warrantyVis;

        // ── OS ────────────────────────────────────────────────
        // Show the feature-update version (e.g. 25H2) inline, in brackets after the edition.
        TxtOs.Text     = string.IsNullOrWhiteSpace(info.WindowsVersion)
                       ? info.OsVersion
                       : $"{info.OsVersion}  ({info.WindowsVersion})";
        TxtKernel.Text = info.KernelVersion;
        TxtArch.Text   = info.KernelArchitecture;

        // ── Performance ───────────────────────────────────────
        TxtCpu.Text = $"{perf.CpuName}  ({perf.Cores} cores / {perf.Threads} threads)";

        // Clock / throttle hint + package power. Only show package power when the sensor actually
        // reports a real value — on machines where LibreHardwareMonitor's driver is blocked (HVCI)
        // it reads 0 W, which is meaningless noise, so hide it rather than show "0 W package".
        var clockBits = new List<string> { perf.ClockText };
        if (temps.CpuPackagePowerW is { } pw && pw >= 1) clockBits.Add($"{pw:F0} W package");
        TxtCpuClock.Text       = string.Join("  ·  ", clockBits);
        TxtCpuClock.Foreground = perf.LikelyThrottled ? StatusColors.Yellow : StatusColors.Muted;

        TxtRam.Text       = $"{perf.RamUsedGb:F1} GB used of {perf.RamTotalGb:F1} GB  ({perf.RamPct:F0}%)";
        BarRam.Value      = perf.RamPct;
        BarRam.Foreground = BarColor(perf.RamPct);

        TxtUptime.Text   = perf.UptimeText;
        TxtLastBoot.Text = perf.BootKnown ? perf.LastBoot.ToString("ddd, d MMM yyyy  HH:mm") : "Unknown";

        // ── Storage (physical disks + volumes) ────────────────
        IcDisks.ItemsSource   = hw.Disks;
        IcVolumes.ItemsSource = hw.Volumes;

        // ── Memory modules ────────────────────────────────────
        IcMemory.ItemsSource = hw.Memory;
        TxtMemSummary.Text   =
            $"{hw.TotalMemoryGb:F0} GB  ·  {hw.SlotsUsed} of {hw.SlotsTotal} slots used"
            + (hw.MaxMemoryGb > 0 ? $"  ·  max {hw.MaxMemoryGb:F0} GB" : "");

        // ── Graphics ──────────────────────────────────────────
        IcGpus.ItemsSource = hw.Gpus;
        if (temps.GpuVramUsedGb is { } vu)
        {
            TxtVram.Text = temps.GpuVramTotalGb is { } vt && vt > 0
                ? $"VRAM in use: {vu:F1} / {vt:F1} GB"
                : $"VRAM in use: {vu:F1} GB";
            TxtVram.Visibility = Visibility.Visible;
        }
        else TxtVram.Visibility = Visibility.Collapsed;

        // ── Monitors ──────────────────────────────────────────
        IcMonitors.ItemsSource  = displays;
        MonitorsSection.Visibility = displays.Count > 0 ? Visibility.Visible : Visibility.Collapsed;

        // ── Firmware ──────────────────────────────────────────
        TxtBios.Text      = hw.BiosVersion;
        TxtBiosDate.Text  = string.IsNullOrEmpty(hw.BiosDate) ? "—" : hw.BiosDate;
        TxtBootMode.Text  = hw.BootMode;
        TxtTpm.Text       = hw.Tpm;
        TxtTpm.Foreground = hw.TpmHealthy == true ? StatusColors.Green
                          : hw.TpmHealthy == false ? StatusColors.Yellow
                          : StatusColors.Muted;

        // ── Battery (laptop only) ─────────────────────────────
        _hasBattery = perf.BatteryPct.HasValue;
        if (perf.BatteryPct.HasValue)
        {
            BatteryCard.Visibility = Visibility.Visible;
            int pct = perf.BatteryPct.Value;
            TxtBattery.Text        = $"{pct}%";
            BarBattery.Value       = pct;
            BarBattery.Foreground  = pct < 20 ? StatusColors.Red
                                   : pct < 50 ? StatusColors.Yellow
                                   : StatusColors.Green;
            // Initial status — the live tick enriches this with charge/discharge rate + runtime.
            TxtBattPower.Text       = perf.OnAcPower == true ? "Plugged in" : "On battery";
            TxtBattPower.Foreground = perf.OnAcPower == true ? StatusColors.Green : StatusColors.Yellow;
            TxtBattVoltage.Text     = "—";   // filled by the live tick's battery poll

            // Wear / capacity, when the battery reports it
            if (hw.BatteryWearPct is { } wear)
            {
                BattHealthSection.Visibility = Visibility.Visible;
                TxtWear.Text       = $"{wear}%";
                TxtWear.Foreground = wear <= 20 ? StatusColors.Green
                                   : wear <= 40 ? StatusColors.Yellow
                                   : StatusColors.Red;
                TxtBattDesign.Text = hw.BatteryDesignMwh is { } d ? $"{d / 1000.0:F1} Wh" : "—";
                TxtBattFull.Text   = hw.BatteryFullMwh   is { } f ? $"{f / 1000.0:F1} Wh" : "—";
            }
            else
            {
                BattHealthSection.Visibility = Visibility.Collapsed;
            }
        }
        else
        {
            BatteryCard.Visibility = Visibility.Collapsed;
        }

        // ── Security ──────────────────────────────────────────
        TxtAv.Text       = $"● {sec.AvName}";
        TxtAv.Foreground = sec.AvRtpEnabled ? StatusColors.Green : StatusColors.Red;

        int fwOff = new[] { sec.FwDomain, sec.FwPrivate, sec.FwPublic }.Count(v => !v);
        TxtFw.Text       = fwOff == 0 ? "● All profiles enabled"
                         : $"● {fwOff} profile(s) disabled";
        TxtFw.Foreground = fwOff == 0 ? StatusColors.Green : StatusColors.Red;

        TxtBl.Text       = $"● {sec.BitLocker}";
        TxtBl.Foreground = sec.BitLocker == "Encrypted" ? StatusColors.Green
                         : sec.BitLocker == "Requires admin" ? StatusColors.Yellow
                         : StatusColors.Red;

        TxtAct.Text       = sec.Activated == true  ? "● Activated"
                          : sec.Activated == false ? "● Not activated"
                          : "● Unknown";
        TxtAct.Foreground = sec.Activated == true ? StatusColors.Green : StatusColors.Red;

        TxtSb.Text       = sec.SecureBoot == true  ? "● Enabled"
                         : sec.SecureBoot == false ? "● Disabled"
                         : "● Unknown";
        TxtSb.Foreground = sec.SecureBoot == true ? StatusColors.Green : StatusColors.Yellow;

        TxtUac.Text       = sec.UacEnabled == true  ? "● Enabled"
                          : sec.UacEnabled == false ? "● Disabled"
                          : "● Unknown";
        TxtUac.Foreground = sec.UacEnabled == true ? StatusColors.Green : StatusColors.Yellow;

        // ── Network ───────────────────────────────────────────
        if (adapter == null)
        {
            TxtNetName.Text     = "";
            NetGrid.Visibility  = Visibility.Collapsed;
            TxtNoNet.Visibility = Visibility.Visible;
        }
        else
        {
            TxtNoNet.Visibility = Visibility.Collapsed;
            NetGrid.Visibility  = Visibility.Visible;
            // This is the primary/default route's adapter — label it so it's clear WHY this one was
            // picked (a machine can have several NICs, VPNs, virtual switches, etc.).
            TxtNetName.Text     = $"{adapter.Name}   (Default)";
            TxtNetType.Text     = adapter.Type;
            TxtNetIp.Text       = adapter.IpAddress;
            TxtNetMask.Text     = adapter.SubnetMask;
            TxtNetGw.Text       = adapter.Gateway;
            TxtNetDns.Text      = adapter.Dns;
            TxtNetMac.Text      = adapter.Mac;
            TxtNetDhcp.Text       = adapter.IpAssignment;
            TxtNetDhcpServer.Text = string.IsNullOrEmpty(adapter.DhcpServer) ? "—" : adapter.DhcpServer;
            TxtNetLease.Text      = adapter.LeaseExpires is { } le ? le.ToString("ddd, d MMM HH:mm") : "—";
        }

        // ── Printers ──────────────────────────────────────────
        IcPrinters.ItemsSource   = printers;
        TxtNoPrinters.Visibility = printers.Count > 0 ? Visibility.Collapsed : Visibility.Visible;

        // ── Local accounts ────────────────────────────────────
        IcAccounts.ItemsSource = accounts;

        // ── Page file / proxy (now under Performance) ─────────
        TxtPageFile.Text  = extras.PageFile;
        TxtProxy.Text     = extras.Proxy;

        // ── Power status (under the Power buttons) ────────────
        TxtPwrSource.Text  = power.PowerSource;
        TxtPwrMode.Text    = power.PowerMode;
        TxtPwrFast.Text    = power.FastStartup;
        TxtPwrHib.Text     = power.Hibernation;
        UpdateHibToggleText();
        TxtPwrSleep.Text   = power.Sleep;
        TxtPwrDisplay.Text = power.DisplayOff;
        TxtPwrWake.Text    = power.LastWake;

        TxtRefreshed.Text = $"Refreshed at {asOf:HH:mm:ss}   ·   Updates every {RefreshSeconds} seconds";
    }

    private static SolidColorBrush BarColor(double pct) =>
        pct >= 90 ? StatusColors.Red :
        pct >= 75 ? StatusColors.Yellow :
        StatusColors.Blue;
}
```
