---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\DiagnosticsPage.xaml.cs
---

# PartnerTool\Pages\DiagnosticsPage.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class DiagnosticsPage : UserControl
{
    private bool _procsLoading;
    private bool _procsLoadedOnce;
    private bool _crashBootLoaded;
    private bool _shownOnce;
    private List<EventEntry> _allEvents = new();       // full 10-day set behind the "More…" view
    private List<HistoryEntry> _allHistory = new();    // full trend behind the health-history "More…" view

    public DiagnosticsPage()
    {
        InitializeComponent();
        IsVisibleChanged += async (_, _) =>
        {
            if (!IsVisible) return;
            if (!_crashBootLoaded) { _crashBootLoaded = true; await LoadCrashBootAsync(); }
            // First view uses the instant startup snapshot; refresh the live data (and the
            // "Refreshed at" stamp) every time the page is re-opened. The manual button is gone.
            if (_shownOnce) { await LoadAsync(); }
            else { _shownOnce = true; if (!_procsLoadedOnce) { _procsLoadedOnce = true; await LoadProcessesAsync(); } }
        };
    }

    private async Task LoadCrashBootAsync()
    {
        var crashTask = Task.Run(CrashInfo.Collect);
        var bootTask  = Task.Run(BootPerfInfo.Collect);
        await Task.WhenAll(crashTask, bootTask);
        PaintCrash(await crashTask);
        PaintBoot(await bootTask);
    }

    private void PaintCrash(CrashInfo c)
    {
        IcBsod.ItemsSource     = c.Bsods;
        TxtNoBsod.Visibility   = c.Bsods.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
        IcAppCrash.ItemsSource = CrashInfo.Group(c.Crashes);
        TxtNoAppCrash.Visibility = c.Crashes.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
    }

    private void PaintBoot(BootPerfInfo b)
    {
        TxtBootLatest.Text = b.LatestBootSeconds is { } s
            ? $"Last boot took {s:F1} seconds"
            : "Last boot duration unavailable";
        IcBootContrib.ItemsSource = b.Contributors;
        TxtNoBoot.Visibility = b.Contributors.Count == 0 && b.BootTimes.Count == 0
            ? Visibility.Visible : Visibility.Collapsed;
    }

    /// <summary>Paint immediately from the startup snapshot (no collection).</summary>
    public void Render(SystemSnapshot snap)
        => Paint(snap.Diagnostics, snap.Reliability, snap.Updates, snap.PowerEvents, snap.CapturedAt);

    private void MoreEvents_Click(object sender, RoutedEventArgs e)
    {
        var rows = _allEvents.Select(ev => new ListRow(
            ev.Message,
            $"{ev.Time:ddd d MMM HH:mm}  ·  {ev.Source}  (ID {ev.Id})",
            ev.Level));
        new ListWindow("Recent Errors — last 10 days", rows) { Owner = Window.GetWindow(this) }.ShowDialog();
    }

    private bool _loading;   // page re-opens trigger a refresh — don't stack them

    private async Task LoadAsync()
    {
        if (_loading) return;   // rapid tab flipping must not pile up WUA-COM reloads
        _loading = true;
        try
        {
            TxtRefreshed.Text = "Loading…";
            var diagTask  = Task.Run(DiagnosticsInfo.Collect);
            var relTask   = Task.Run(ReliabilityInfo.CollectIndex);   // index only; records are on-demand
            var updTask   = Task.Run(UpdateHistoryInfo.Collect);
            var powerTask = Task.Run(() => PowerEventsInfo.Collect());
            await Task.WhenAll(diagTask, relTask, updTask, powerTask);
            Paint(await diagTask, await relTask, await updTask, await powerTask, DateTime.Now);
            await LoadProcessesAsync();
        }
        catch (Exception ex)
        {
            // A collector fault must not escalate to the global "Something went wrong" dialog.
            TxtRefreshed.Text = $"Last refresh failed at {DateTime.Now:HH:mm:ss} — {ex.Message}";
        }
        finally { _loading = false; }
    }

    private void Paint(DiagnosticsInfo d, ReliabilityInfo rel, UpdateHistoryInfo upd,
                       List<PowerEvent> power, DateTime asOf)
    {
        // ── Reliability ──────────────────────────────────────
        if (rel.StabilityIndex is { } si)
        {
            TxtStability.Text       = $"{si:F1} / 10";
            TxtStability.Foreground = si >= 8 ? StatusColors.Green : si >= 5 ? StatusColors.Yellow : StatusColors.Red;
        }
        else
        {
            TxtStability.Text       = "—";
            TxtStability.Foreground = StatusColors.Muted;
        }
        // The index looks alarmingly low on machines that just install/update a lot even when they
        // never crash — Windows counts every install, update and driver change against it. Say so,
        // and show the recent peak as reassurance that the machine can and does score well.
        var note = "Windows lowers this for app installs, updates and driver changes — not only crashes. " +
                   "See Crash History below for actual stability.";
        if (rel.RecentPeak is { } pk && (rel.StabilityIndex is not { } cur || pk > cur + 0.05))
            note = $"7-day best: {pk:F1}/10.   " + note;
        TxtStabilityNote.Text = note;

        // ── Power & restart history ──────────────────────────
        IcPowerEvents.ItemsSource = power;
        TxtNoPower.Visibility     = power.Count == 0 ? Visibility.Visible : Visibility.Collapsed;

        // ── Crash dumps ──────────────────────────────────────
        if (d.MinidumpCount == 0 && !d.MemoryDump)
        {
            TxtDumps.Text       = "● None on disk (any older dumps were cleaned up)";
            TxtDumps.Foreground = StatusColors.Green;
        }
        else
        {
            var parts = new List<string>();
            if (d.MinidumpCount > 0)
                parts.Add($"{d.MinidumpCount} minidump(s)" +
                          (d.LatestDump is { } dt ? $", latest {dt:ddd d MMM HH:mm}" : ""));
            if (d.MemoryDump) parts.Add("full MEMORY.DMP present");
            TxtDumps.Text       = "● " + string.Join("  ·  ", parts);
            TxtDumps.Foreground = StatusColors.Yellow;
        }

        // ── Device problems ──────────────────────────────────
        IcDevices.ItemsSource     = d.Devices;
        IcDevices.Visibility      = d.Devices.Count == 0 ? Visibility.Collapsed : Visibility.Visible;
        TxtNoDevices.Visibility   = d.Devices.Count == 0 ? Visibility.Visible : Visibility.Collapsed;

        // ── Recent errors (show last 10; "More…" opens the full 10-day list) ──
        _allEvents = d.Events;
        IcEvents.ItemsSource     = d.Events.Take(10).ToList();
        IcEvents.Visibility      = d.Events.Count == 0 ? Visibility.Collapsed : Visibility.Visible;
        TxtNoEvents.Visibility   = d.Events.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
        BtnMoreEvents.Visibility = d.Events.Count > 10 ? Visibility.Visible : Visibility.Collapsed;

        // ── Health history (last 7 days here; "More…" opens the full trend) ──
        _allHistory = SnapshotHistory.Load().OrderByDescending(h => h.When).ToList();
        IcHistory.ItemsSource    = _allHistory.Take(7).ToList();
        TxtNoHistory.Visibility  = _allHistory.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
        BtnMoreHistory.Visibility = _allHistory.Count > 7 ? Visibility.Visible : Visibility.Collapsed;

        TxtRefreshed.Text = $"Refreshed at {asOf:HH:mm:ss}";
    }

    // ── Reliability records (on demand — the WMI class can take ~a minute) → shown in a popup ──
    private bool _relRecordsLoading;

    private async void LoadReliability_Click(object sender, RoutedEventArgs e)
    {
        if (_relRecordsLoading) return;
        _relRecordsLoading = true;
        var label = BtnLoadReliability.Content;
        BtnLoadReliability.IsEnabled = false;
        BtnLoadReliability.Content   = "Loading…";
        try
        {
            var records = await Task.Run(ReliabilityInfo.CollectRecords);
            if (records.Count == 0)
            {
                MessageWindow.Show("Reliability History", "No detailed records",
                    "Windows returned no reliability records for this machine.",
                    MessageKind.Info, Window.GetWindow(this));
                return;
            }
            var rows = records.Select(r => new ListRow(
                r.Message,
                $"{r.Time:ddd d MMM HH:mm}  ·  {r.Source}",
                "")).ToList();
            new ListWindow("Reliability History (detailed)", rows) { Owner = Window.GetWindow(this) }.ShowDialog();
        }
        finally { _relRecordsLoading = false; BtnLoadReliability.IsEnabled = true; BtnLoadReliability.Content = label; }
    }

    // Full health-history trend (the card shows only the last 7 days).
    private void MoreHistory_Click(object sender, RoutedEventArgs e)
    {
        var rows = _allHistory.Select(h => new ListRow(
            h.When.ToString("dddd, d MMM yyyy"),
            $"Disk free {h.DiskFreeGb:F0} GB   ·   RAM {h.RamPct:F0}%"
              + (h.BatteryWearPct is { } w ? $"   ·   Batt wear {w}%" : "")
              + (h.StabilityIndex is { } s ? $"   ·   Stability {s:F1}/10" : ""),
            "")).ToList();
        new ListWindow("Health History — all recorded days", rows) { Owner = Window.GetWindow(this) }.ShowDialog();
    }

    // ── Live processes ───────────────────────────────────────
    private async void ProcRefresh_Click(object sender, RoutedEventArgs e) => await LoadProcessesAsync();

    private async Task LoadProcessesAsync()
    {
        if (_procsLoading) return;
        _procsLoading = true;
        BtnProcRefresh.IsEnabled = false;
        try { IcProcesses.ItemsSource = (await ProcessInfo.TopAsync()).Take(10).ToList(); }
        finally { _procsLoading = false; BtnProcRefresh.IsEnabled = true; }
    }

    private async void KillProc_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: int pid }) return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        var row = (sender as Button)?.DataContext as ProcInfo;
        var name = row?.Name ?? pid.ToString();

        // Ending a critical Windows process forces a BSOD — block it outright.
        if (ProcessInfo.IsCritical(name))
        {
            MessageWindow.Show("End Process", $"“{name}” is a critical Windows process",
                "Ending it would immediately crash Windows (CRITICAL_PROCESS_DIED), so it's blocked.",
                MessageKind.Warning, Window.GetWindow(this));
            return;
        }

        if (!MessageWindow.Confirm("End Process", $"End “{name}” (PID {pid})?",
                "Ending a process can cause unsaved work to be lost or a service to stop. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Process", $"End process {name} (PID {pid})");
        if (ProcessInfo.TryKill(pid))
        {
            ActivityLog.Result("Process", "ended");
            await LoadProcessesAsync();
        }
        else
        {
            ActivityLog.Result("Process", "failed — access denied or already exited");
            MessageWindow.Show("End Process", "Couldn't end the process",
                "Access was denied or it has already exited.", MessageKind.Error, Window.GetWindow(this));
        }
    }
}
```
