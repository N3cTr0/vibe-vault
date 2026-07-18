---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\RepairPage.xaml.cs
---

# PartnerTool\Pages\RepairPage.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Text;
using System.Threading;
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class RepairPage : UserControl
{
    private bool _busy;
    private bool _restoreLoaded;
    private bool _dellLoaded;

    public RepairPage()
    {
        InitializeComponent();
        TxtProxyStatus.Text = ProxyRepair.CurrentState();
        IsVisibleChanged += (_, _) =>
        {
            if (!IsVisible) return;
            TxtProxyStatus.Text = ProxyRepair.CurrentState();
            if (!_restoreLoaded) { _restoreLoaded = true; LoadRestorePoints(); }
            if (!_dellLoaded) { _dellLoaded = true; ShowDellCardIfApplicable(); }
        };
    }

    // ── Dell SupportAssist / VSS shadow storage (Dell hardware only) ──────

    /// <summary>
    /// Decide whether the Dell card should appear — a cheap manufacturer + folder-exists check, no
    /// directory walk. The actual sizing waits for the tech to click Scan, so opening the Repair
    /// page never kicks off a multi-GB folder walk on its own.
    /// </summary>
    private async void ShowDellCardIfApplicable()
    {
        bool applicable;
        try { applicable = await Task.Run(DellRemediation.IsApplicable); }
        catch { return; }
        if (!applicable) return;   // leave the card collapsed on non-Dell machines

        CardDell.Visibility = Visibility.Visible;
        BtnDellRefresh.Content = "Scan";
        TxtDellStatus.Text = "Not scanned yet — click Scan to size SARemediation and check shadow storage on C:.";
        TxtDellStatus.Foreground = StatusColors.Muted;
    }

    /// <summary>
    /// Size up SARemediation and check whether VSS is bounded. Sizing the folder walks a large tree,
    /// so it runs off the UI thread. Only ever called from the Scan/Refresh button, never on load.
    /// </summary>
    private async Task LoadDellAsync()
    {
        DellRemediation.DellInfo info;
        try { info = await Task.Run(DellRemediation.Collect); }
        catch { return; }

        if (!info.IsDell || !info.FolderExists) return;   // leave the card collapsed

        CardDell.Visibility = Visibility.Visible;
        BtnDellRefresh.Content = "Refresh";   // first scan done — the button now re-scans
        _dellVssUnbounded = info.VssUnbounded;
        BtnDellCapVss.IsEnabled = info.VssUnbounded;

        var parts = new List<string> { $"SARemediation is using {info.TotalGb:F1} GB" };
        if (info.SnapshotBytes > 0) parts.Add($"{info.SnapshotGb:F1} GB of that is repair snapshots");
        parts.Add(info.VssUnbounded
            ? $"shadow storage on C: is UNBOUNDED ({info.VssUsedGb:F1} GB used)"
            : info.VssConfigured
                ? $"shadow storage on C: is capped at {info.VssMaxText}"
                : $"shadow storage on C: is {info.VssMaxText}");

        TxtDellStatus.Text = string.Join("  ·  ", parts);
        TxtDellStatus.Foreground = info.VeryBad || info.VssUnbounded ? StatusColors.Yellow
                                 : info.Bloated ? StatusColors.Yellow
                                 : StatusColors.Green;
    }

    private async void DellRefresh_Click(object sender, RoutedEventArgs e)
    {
        BtnDellRefresh.IsEnabled = false;
        TxtDellStatus.Text = "Sizing C:\\ProgramData\\Dell\\SARemediation…";
        TxtDellStatus.Foreground = StatusColors.Yellow;
        try { await LoadDellAsync(); }
        finally { BtnDellRefresh.IsEnabled = true; }
    }

    private void DellOpenSa_Click(object sender, RoutedEventArgs e) => DellRemediation.OpenSupportAssistSettings();

    private async void DellCapVss_Click(object sender, RoutedEventArgs e)
    {
        // Shrinking the shadow-storage limit discards existing restore points — confirm, then gate.
        if (!MessageWindow.Confirm("Shadow Storage",
                $"Cap VSS shadow storage on C: at {DellRemediation.VssMaxPercent}%?",
                "This is Dell's own fix for unbounded shadow copies (KB 000129138). Existing System Restore " +
                "points that don't fit under the new limit will be deleted.",
                MessageKind.Warning, Window.GetWindow(this)))
            return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!BeginBusy()) return;

        UseLog(DellLogScroll, DellLog);
        Log($"▶ Cap VSS shadow storage on C: at {DellRemediation.VssMaxPercent}%");
        TxtDellStatus.Foreground = StatusColors.Yellow;
        TxtDellStatus.Text = "Resizing shadow storage…";
        try
        {
            var result = await DellRemediation.BoundVssAsync(Log);
            Log($"━━━ {result} ━━━");
            await LoadDellAsync();
        }
        catch (Exception ex)
        {
            Log($"Error: {ex.Message}");
            TxtDellStatus.Text = ex.Message;
            TxtDellStatus.Foreground = StatusColors.Red;
        }
        finally { EndBusy(); }
    }

    // ── Proxy / WPAD auto-detect (Outlook/Office connectivity) ────────────
    private async void FixWpad_Click(object sender, RoutedEventArgs e)
    {
        ActivityLog.Action("Proxy", "Re-enable WPAD auto-detect (toggle off→on, HKCU DefaultConnectionSettings)");
        BtnFixWpad.IsEnabled = false;
        TxtProxyStatus.Text = "Toggling auto-detect…";
        TxtProxyStatus.Foreground = StatusColors.Yellow;
        // ResetAutoDetect sleeps ~300ms between the off/on toggle — run it off the UI thread.
        var (ok, msg) = await Task.Run(ProxyRepair.ResetAutoDetect);
        ActivityLog.Result("Proxy", ok ? "done" : msg);
        TxtProxyStatus.Text = msg;
        TxtProxyStatus.Foreground = ok ? StatusColors.Green : StatusColors.Red;
        BtnFixWpad.IsEnabled = true;
    }

    private async void ResetWinhttp_Click(object sender, RoutedEventArgs e)
    {
        // Resetting the WinHTTP proxy to direct can break connectivity in proxied environments.
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        BtnResetWinhttp.IsEnabled = false;
        TxtProxyStatus.Text = "Resetting WinHTTP proxy…";
        TxtProxyStatus.Foreground = StatusColors.Yellow;
        try
        {
            ActivityLog.Command("Proxy", "netsh.exe", "winhttp reset proxy");
            var outp = await ProcessRunner.RunCaptureAsync("netsh.exe", "winhttp reset proxy", 15000);
            var line = outp.Replace("\r", " ").Replace("\n", " ").Trim();
            TxtProxyStatus.Text = $"WinHTTP proxy reset to direct.  {line}   ·   {ProxyRepair.CurrentState()}";
            TxtProxyStatus.Foreground = StatusColors.Green;
        }
        catch (Exception ex) { TxtProxyStatus.Text = ex.Message; TxtProxyStatus.Foreground = StatusColors.Red; }
        finally { BtnResetWinhttp.IsEnabled = true; }
    }

    // ── Clean temp files (all users) ──────────────────────────────────────
    // Scan is a read-only dry-run (no gate) — shows how much the targets hold before cleaning.
    private async void ScanTemp_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        TxtCleanTempStatus.Foreground = StatusColors.Yellow;
        TxtCleanTempStatus.Text = "Scanning…";
        UseLog(CleanTempLogScroll, CleanTempLog);
        Log("▶ Scan temp files (all users) — nothing is deleted");
        try
        {
            void Status(string s) => Dispatcher.Invoke(() => TxtCleanTempStatus.Text = s);
            var result = await Task.Run(() => TempCleaner.Scan(Status));
            foreach (var line in result.Lines) Log(line);
            Log($"━━━ {result.Summary} ━━━");
            TxtCleanTempStatus.Text = result.FileCount == 0
                ? "Nothing to clean — the temp folders are already empty."
                : $"{result.Summary}  Click Clean to delete.";
            TxtCleanTempStatus.Foreground = StatusColors.Green;
        }
        catch (Exception ex)
        {
            Log($"Error: {ex.Message}");
            TxtCleanTempStatus.Text = ex.Message; TxtCleanTempStatus.Foreground = StatusColors.Red;
        }
        finally { EndBusy(); SaveLog("CleanTemp"); }
    }

    private async void CleanTemp_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!MessageWindow.Confirm("Clean Temp Files", "Delete temp files for all users?",
                "This permanently deletes the contents of the temp and cache folders for every user " +
                "profile and the system temp. Files in use are skipped; the folders themselves stay. " +
                "Continue?", MessageKind.Warning, Window.GetWindow(this)))
            return;

        if (!BeginBusy()) return;   // page-wide lock — don't let another heavy fix run concurrently
        TxtCleanTempStatus.Foreground = StatusColors.Yellow;
        TxtCleanTempStatus.Text = "Cleaning…";
        UseLog(CleanTempLogScroll, CleanTempLog);
        Log("▶ Clean temp files (all users)");
        ActivityLog.Action("Repair", "Clean temp files (all users)");
        try
        {
            void Status(string s) => Dispatcher.Invoke(() => TxtCleanTempStatus.Text = s);
            var result = await Task.Run(() => TempCleaner.Clean(Status, Log));
            Log($"━━━ {result.Summary} ━━━");
            ActivityLog.Result("Repair", result.Summary);
            TxtCleanTempStatus.Text = result.Summary;
            TxtCleanTempStatus.Foreground = StatusColors.Green;
        }
        catch (Exception ex)
        {
            Log($"Error: {ex.Message}");
            TxtCleanTempStatus.Text = ex.Message; TxtCleanTempStatus.Foreground = StatusColors.Red;
        }
        finally { EndBusy(); SaveLog("CleanTemp"); }
    }

    // ── busy / button gating ──────────────────────────────────────────────

    /// <summary>
    /// BeginBusy + the app-wide ServicingLock, for fixes that drive CBS/TrustedInstaller
    /// (DISM, SFC, CHKDSK, Windows Update work) — so they can't overlap Update All (or each
    /// other across pages). Pair with <see cref="EndServicing"/> in the handler's finally.
    /// </summary>
    private bool BeginServicing(string operation)
    {
        if (!ServicingLock.TryAcquire(operation))
        {
            MessageWindow.Show("Please Wait", "Another heavy operation is running",
                $"“{ServicingLock.CurrentOperation}” is still running. Running two Windows " +
                "servicing operations at once makes them fail — wait for it to finish first.",
                MessageKind.Warning, Window.GetWindow(this));
            return false;
        }
        if (!BeginBusy()) { ServicingLock.Release(); return false; }
        return true;
    }

    private void EndServicing()
    {
        EndBusy();
        ServicingLock.Release();
    }

    private bool BeginBusy()
    {
        if (_busy) return false;
        _busy = true;
        SetButtons(false);
        return true;
    }

    private void EndBusy()
    {
        _busy = false;
        SetButtons(true);
    }

    private void SetButtons(bool on)
    {
        BtnFullRepair.IsEnabled = on;
        BtnChkdsk.IsEnabled     = on;
        BtnWuReset.IsEnabled    = on;
        BtnCleanup.IsEnabled    = on;
        BtnRestorePoint.IsEnabled   = on;
        BtnEmptyBin.IsEnabled       = on;
        BtnRestartSpooler.IsEnabled = on;
        BtnClearQueue.IsEnabled     = on;
        BtnRestartExplorer.IsEnabled = on;
        BtnRestartAudio.IsEnabled    = on;
        BtnReregStore.IsEnabled      = on;
        BtnClearIconCache.IsEnabled  = on;
        BtnMemDiag.IsEnabled         = on;
        BtnSchedChkdsk.IsEnabled     = on;
        BtnBatteryReport.IsEnabled   = on;
        BtnGpReport.IsEnabled        = on;
        BtnCollectDiag.IsEnabled     = on;
        BtnRemoveOfficeLangs.IsEnabled = on;
        BtnScanInstaller.IsEnabled   = on;
        BtnCleanInstaller.IsEnabled  = on;
        BtnAdobePatchFix.IsEnabled   = on;
        BtnScanTemp.IsEnabled        = on;
        BtnCleanTemp.IsEnabled       = on;
        BtnScanFeatUpd.IsEnabled     = on;
        BtnCleanFeatUpd.IsEnabled    = on;
        BtnFixWpad.IsEnabled         = on;
        BtnResetWinhttp.IsEnabled    = on;
        BtnDellRefresh.IsEnabled     = on;
        BtnDellOpenSa.IsEnabled      = on;
        // Capping shadow storage only makes sense while it's actually unbounded.
        BtnDellCapVss.IsEnabled      = on && CardDell.Visibility == Visibility.Visible && _dellVssUnbounded;
    }

    private bool _dellVssUnbounded;

    // ── logging (per-section: each action points Log() at its own inline log) ──

    private TextBlock?    _logText;
    private ScrollViewer? _logScroll;

    // Point logging at a section's inline log, reveal it, and clear it for a fresh run.
    private void UseLog(ScrollViewer scroll, TextBlock text)
    {
        _logScroll = scroll;
        _logText   = text;
        if (scroll.Parent is UIElement panel) panel.Visibility = Visibility.Visible;
        text.Text = "";
    }

    private void Log(string line) => Dispatcher.Invoke(() =>
    {
        if (_logText is null) return;   // no active section log — output dropped (e.g. Reports)
        _logText.Text += line + "\n";
        _logScroll?.ScrollToBottom();
    });

    private void SaveLog(string tag)
    {
        try
        {
            var body = Dispatcher.Invoke(() => _logText?.Text) ?? "";
            if (string.IsNullOrWhiteSpace(body)) return;

            const string logDir = @"C:\PCI\Logs";
            Directory.CreateDirectory(logDir);
            var path = Path.Combine(logDir,
                $"PartnerTool_{tag}_{Environment.MachineName}_{DateTime.Now:MMddyyyy_HHmmss}.log");   // seconds — a Scan then Clean in the same minute must not overwrite each other

            var sb = new StringBuilder();
            sb.AppendLine($"PARTNER TOOL — {tag.ToUpperInvariant()} LOG");
            sb.AppendLine($"Device    : {Environment.MachineName}");
            sb.AppendLine($"User      : {Environment.UserName}");
            sb.AppendLine($"Date/Time : {DateTime.Now:MM-dd-yyyy HH:mm:ss}");
            sb.AppendLine(new string('─', 50));
            sb.AppendLine();
            sb.AppendLine(body);

            File.WriteAllText(path, LogText.ToAscii(sb.ToString()), LogText.Utf8NoBom);   // plain ASCII → no mojibake in tickets
            Log($"Log saved → {path}");
        }
        catch (Exception ex)
        {
            Log($"Could not save log: {ex.Message}");
        }
    }

    // ── generic "run a process with live progress + elapsed heartbeat" ─────

    private async Task<(int code, string text)> RunWithProgress(
        string exe, string args, Encoding? enc, string startLog, Action<string> setStatus)
    {
        Log($"▶ {startLog}");
        ActivityLog.Action("Repair", startLog);
        var sw       = Stopwatch.StartNew();
        int lastPct  = -1;
        using var cts = new CancellationTokenSource();

        // Heartbeat: tick the status every second so a silent long step never looks frozen.
        var ticker = Task.Run(async () =>
        {
            try
            {
                while (true)
                {
                    await Task.Delay(1000, cts.Token);
                    int p = lastPct; TimeSpan el = sw.Elapsed;
                    Dispatcher.Invoke(() => setStatus(
                        p >= 0 ? $"Running… {p}%  ({el:mm\\:ss})"
                               : $"Running… ({el:mm\\:ss})"));
                }
            }
            catch (OperationCanceledException) { }
        });

        var sb = new StringBuilder();
        int code;
        try
        {
            code = await ProcessRunner.RunAsync(exe, args, enc,
                line => { lock (sb) { sb.AppendLine(line); } Log($"  {line}"); },
                pct  => lastPct = pct);
        }
        finally
        {
            cts.Cancel();
            try { await ticker; } catch { }
        }
        return (code, sb.ToString());
    }

    private Task<(int, string)> RunStep(UpdateTask task, Encoding? enc, string exe, string args, string startLog)
    {
        task.StatusColor = StatusColors.Yellow;
        return RunWithProgress(exe, args, enc, startLog, s => task.StatusText = s);
    }

    private async Task<int> RunUtil(string exe, string args, Encoding? enc, string startLog, TextBlock status)
    {
        var (code, _) = await RunWithProgress(exe, args, enc, startLog, s => status.Text = s);
        return code;
    }

    private async Task<int> RunPowerShell(string fileName, string content, string startLog, TextBlock status)
    {
        var script = Path.Combine(Path.GetTempPath(), fileName);
        try
        {
            await File.WriteAllTextAsync(script, content);
            return await RunUtil("powershell.exe",
                $"-ExecutionPolicy Bypass -NonInteractive -File \"{script}\"", null, startLog, status);
        }
        finally { try { File.Delete(script); } catch { } }
    }

    private static void Set(TextBlock tb, string text, System.Windows.Media.Brush color)
    {
        tb.Text = text;
        tb.Foreground = color;
    }

    private static void Set(UpdateTask task, string text, System.Windows.Media.Brush color)
    {
        task.StatusText  = text;
        task.StatusColor = color;
    }

    // ── FULL REPAIR ───────────────────────────────────────────────────────

    private async void FullRepair_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!BeginServicing("System File & Image Repair")) return;
        BtnFullRepair.Content = "Running…";
        UseLog(FullRepairLogScroll, FullRepairLog);

        var tCheck   = new UpdateTask { Name = "DISM — CheckHealth" };
        var tScan    = new UpdateTask { Name = "DISM — ScanHealth" };
        var tRestore = new UpdateTask { Name = "DISM — RestoreHealth" };
        var tSfc     = new UpdateTask { Name = "SFC — /scannow" };

        TaskList.ItemsSource = new[] { tCheck, tScan, tRestore, tSfc };
        PnlTasks.Visibility  = Visibility.Visible;

        try
        {
            // DISM first, then SFC — SFC repairs system files from the component store
            // that DISM restores, so the store must be healthy before SFC runs.
            await RunDismScan(tCheck, "/CheckHealth", "DISM CheckHealth (quick)");
            await RunDismScan(tScan,  "/ScanHealth",  "DISM ScanHealth (full scan)");
            await RunDismRestore(tRestore);
            await RunSfc(tSfc);
            Log("━━━ Full repair complete ━━━");
        }
        catch (Exception ex)
        {
            Log($"━━━ Stopped — unexpected error: {ex.Message} ━━━");
        }
        finally
        {
            BtnFullRepair.Content = "Run Again";
            EndServicing();
            SaveLog("Repair");
        }
    }

    private async Task RunDismScan(UpdateTask task, string op, string label)
    {
        var (code, text) = await RunStep(task, null, "dism.exe", $"/Online /Cleanup-Image {op}", label);
        string t = text.ToLowerInvariant();
        if (code != 0 && code != 3010)
            Set(task, "● Error (check log)", StatusColors.Red);
        else if (t.Contains("repairable"))
            Set(task, "● Corruption found — repairable", StatusColors.Yellow);
        else if (t.Contains("not repairable"))
            Set(task, "● Corruption — not repairable", StatusColors.Red);
        else
            Set(task, "● No corruption", StatusColors.Green);
    }

    private async Task RunDismRestore(UpdateTask task)
    {
        var (code, text) = await RunStep(task, null, "dism.exe",
            "/Online /Cleanup-Image /RestoreHealth", "DISM RestoreHealth (repairing from Windows Update)");
        string t = text.ToLowerInvariant();

        if (t.Contains("0x800f081f") || t.Contains("source files could not be found"))
            Set(task, "● Failed — repair source unavailable", StatusColors.Red);
        else if (code == 3010)
            Set(task, "● Repaired — reboot required", StatusColors.Yellow);
        else if (code == 0)
            Set(task, t.Contains("corruption was repaired") ? "● Repaired" : "● Healthy", StatusColors.Green);
        else
            Set(task, "● Done (check log)", StatusColors.Yellow);
    }

    private async Task RunSfc(UpdateTask task)
    {
        // sfc.exe writes UTF-16 to stdout — decode it as Unicode so result text is readable.
        var (_, text) = await RunStep(task, Encoding.Unicode, "sfc.exe", "/scannow", "SFC /scannow");
        string t = text.ToLowerInvariant();

        if (t.Contains("did not find any integrity"))
            Set(task, "● No violations found", StatusColors.Green);
        else if (t.Contains("successfully repaired"))
            Set(task, "● Repaired", StatusColors.Green);
        else if (t.Contains("unable to fix"))
            Set(task, "● Some unfixable (check log)", StatusColors.Red);
        else if (t.Contains("could not perform"))
            Set(task, "● Could not run", StatusColors.Red);
        else
            Set(task, "● Done (check log)", StatusColors.Yellow);
    }

    // ── CHECK DISK (CHKDSK) — standalone ──────────────────────────────────

    private async void Chkdsk_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!BeginServicing("Check Disk (CHKDSK scan)")) return;
        UseLog(ChkdskLogScroll, ChkdskLog);
        Set(TxtChkdskStatus, "Running…", StatusColors.Yellow);
        try
        {
            var code = await RunUtil("chkdsk.exe", "C: /scan", null, "CHKDSK C: /scan (online)", TxtChkdskStatus);
            Set(TxtChkdskStatus, code == 0 ? "● No problems found" : "● Problems found — run chkdsk /f /r",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        finally { EndServicing(); SaveLog("Chkdsk"); }
    }

    // ── WINDOWS INSTALLER CLEANUP (Adobe bloat) ───────────────────────────

    private async void ScanInstaller_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        Set(TxtInstallerStatus, @"Scanning C:\Windows\Installer (reading each package)…", StatusColors.Yellow);
        try
        {
            var r = await Task.Run(InstallerCleanup.Scan);
            if (!r.KeepSetValid)
            {
                Set(TxtInstallerStatus, "● " + (r.Error ?? "Scan failed."), StatusColors.Red);
                return;
            }
            if (r.Orphans.Count == 0)
            {
                Set(TxtInstallerStatus,
                    $"● No orphaned installer files — {r.ReferencedCount} referenced package(s), nothing to clean.",
                    StatusColors.Green);
                return;
            }

            // Dry-run preview: list what WOULD be removed + what each package is for.
            UseLog(InstallerLogScroll, InstallerLog);
            Log($"▶ Installer scan (preview — nothing deleted) — {r.Orphans.Count} orphaned file(s), {r.OrphanGb:F1} GB");
            LogInstallerBreakdown(r);
            Log("  ── files ──");
            foreach (var o in r.Orphans) Log("  " + InstallerCleanup.Line("would remove", o));
            Log($"━━━ {r.Orphans.Count} orphaned file(s), {r.OrphanGb:F1} GB — review, then click Clean ━━━");
            SaveLog("InstallerScan");

            Set(TxtInstallerStatus,
                $"● {r.Orphans.Count} orphaned file(s) using {r.OrphanGb:F1} GB can be freed  ·  {r.ReferencedCount} referenced package(s) kept. See the log below for the full list.",
                StatusColors.Yellow);
        }
        catch (Exception ex) { Set(TxtInstallerStatus, "● " + ex.Message, StatusColors.Red); }
        finally { EndBusy(); }
    }

    private void LogInstallerBreakdown(InstallerScanResult r)
    {
        Log("  ── by product ──");
        foreach (var (product, count, bytes) in r.TopProducts(12))
            Log($"  {product} — {count} file(s), {bytes / 1073741824.0:F1} GB");
    }

    private async void CleanInstaller_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        try
        {
            Set(TxtInstallerStatus, "Scanning…", StatusColors.Yellow);
            var r = await Task.Run(InstallerCleanup.Scan);
            if (!r.KeepSetValid)
            {
                Set(TxtInstallerStatus, "● " + (r.Error ?? "Scan failed — aborted for safety."), StatusColors.Red);
                return;
            }
            if (r.Orphans.Count == 0)
            {
                Set(TxtInstallerStatus, "● No orphaned files to remove.", StatusColors.Green);
                return;
            }

            if (!TechGate.Verify(Window.GetWindow(this))) return;
            var breakdown = string.Join("\n", r.TopProducts(8)
                .Select(p => $"   • {p.Product} — {p.Count} file(s), {p.Bytes / 1073741824.0:F1} GB"));
            if (!MessageWindow.Confirm("Installer Cleanup",
                    $"Delete {r.Orphans.Count} orphaned installer file(s) — freeing {r.OrphanGb:F1} GB?",
                    $"Mostly:\n{breakdown}\n\nThese files in C:\\Windows\\Installer are not referenced by any installed product " +
                    $"(checked against Windows Installer's own database), so removing them is safe and won't affect the repair " +
                    $"or uninstall of anything installed. {r.ReferencedCount} referenced package(s) will be kept. The full list is " +
                    $"in the log. This cannot be undone. Continue?",
                    MessageKind.Warning, Window.GetWindow(this)))
                return;

            UseLog(InstallerLogScroll, InstallerLog);
            Log($"▶ Installer cleanup — {r.Orphans.Count} orphaned file(s), {r.OrphanGb:F1} GB");
            LogInstallerBreakdown(r);
            Log("  ── files ──");
            ActivityLog.Action("Repair", $"Clean {r.Orphans.Count} orphaned installer file(s) (~{r.OrphanGb:F1} GB) from C:\\Windows\\Installer");
            Set(TxtInstallerStatus, "Cleaning…", StatusColors.Yellow);

            var (deleted, freed, failed) = await Task.Run(() => InstallerCleanup.DeleteOrphans(r.Orphans, l => Log("  " + l)));
            var summary = $"Removed {deleted} file(s), freed {freed / 1073741824.0:F1} GB"
                        + (failed > 0 ? $"; {failed} in use/skipped." : ".");
            Log($"━━━ {summary} ━━━");
            ActivityLog.Result("Repair", summary);
            Set(TxtInstallerStatus, "● " + summary, StatusColors.Green);
        }
        catch (Exception ex)
        {
            Log($"Error: {ex.Message}");
            Set(TxtInstallerStatus, "● " + ex.Message, StatusColors.Red);
        }
        finally { EndBusy(); SaveLog("InstallerCleanup"); }
    }

    private void AdobePatchFix_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;   // writes an HKLM policy value
        ActivityLog.Action("Repair", "Apply Adobe PatchCleanFlag fix (HKLM Policies\\Adobe FeatureLockdown)");
        var (ok, msg) = InstallerCleanup.ApplyAdobePatchCleanFix();
        ActivityLog.Result("Repair", ok ? "applied" : msg);
        Set(TxtInstallerStatus, "● " + msg, ok ? StatusColors.Green : StatusColors.Red);
    }

    // ── OFFICE LANGUAGE PACKS ─────────────────────────────────────────────

    private async void RemoveOfficeLangs_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        try
        {
            Set(TxtOfficeLangStatus, "Scanning Office languages…", StatusColors.Yellow);
            var scan = await Task.Run(OfficeLanguages.Scan);

            if (!scan.OfficeFound)
            {
                Set(TxtOfficeLangStatus, "● No Click-to-Run Office detected on this PC.", StatusColors.Yellow);
                return;
            }

            var nonEng = scan.NonEnglish;
            var keepList = string.Join(", ", scan.English.Select(p => p.Culture));

            if (nonEng.Count == 0)
            {
                Set(TxtOfficeLangStatus,
                    $"● Only English installed ({keepList}). Nothing to remove.", StatusColors.Green);
                return;
            }
            if (scan.English.Count == 0)
            {
                // Never strip the last language — Office would be left with none.
                Set(TxtOfficeLangStatus,
                    "● No English pack is installed — not removing the others (Office would have no language). " +
                    "Add English first.", StatusColors.Red);
                return;
            }

            // Read-only scan is free; require the tech code before changing anything.
            if (!TechGate.Verify(Window.GetWindow(this))) return;

            var removeList = string.Join("\n", nonEng.Select(p => $"   • {p.FriendlyName}  ({p.Culture})"));
            if (!MessageWindow.Confirm("Remove Office Languages",
                    $"Remove {nonEng.Count} non-English Office language pack(s)?",
                    $"Will remove:\n{removeList}\n\nKeeping English: {keepList}\n\n" +
                    "All Office apps (Outlook, Word, Excel…) will be closed during removal. Continue?",
                    MessageKind.Warning, Window.GetWindow(this)))
                return;

            UseLog(OfficeLangLogScroll, OfficeLangLog);
            Log($"━━━ Removing {nonEng.Count} Office language pack(s), keeping English ({keepList}) ━━━");
            ActivityLog.Action("Repair", $"Remove {nonEng.Count} non-English Office language pack(s), keeping English ({keepList})");
            int removed = 0, failed = 0;

            foreach (var pack in nonEng)
            {
                var (exe, args) = OfficeLanguages.BuildSilentRemoval(pack.UninstallString);
                var (code, _) = await RunWithProgress(exe, args, null,
                    $"Removing {pack.FriendlyName} ({pack.Culture})",
                    s => Set(TxtOfficeLangStatus, $"Removing {pack.FriendlyName}… {s}", StatusColors.Yellow));

                // Confirm against the registry rather than trusting the exit code.
                bool gone = await Task.Run(() =>
                    !OfficeLanguages.Scan().Packs.Any(p => p.Culture.Equals(pack.Culture, StringComparison.OrdinalIgnoreCase)));
                if (gone) { removed++; Log($"  ✓ {pack.Culture} removed"); }
                else      { failed++;  Log($"  ✗ {pack.Culture} still present (uninstaller exit {code})"); }
            }

            Log("━━━ Office language cleanup complete ━━━");
            Set(TxtOfficeLangStatus,
                failed == 0
                    ? $"● Done — removed {removed} language pack(s); kept English ({keepList})."
                    : $"● Removed {removed}, {failed} still present (check log) — may need a retry or a reboot.",
                failed == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        catch (Exception ex)
        {
            Set(TxtOfficeLangStatus, $"● Error: {ex.Message}", StatusColors.Red);
        }
        finally { EndBusy(); SaveLog("OfficeLang"); }
    }

    // ── FEATURE-UPDATE LEFTOVERS (Windows.old etc.) ───────────────────────

    private async void ScanFeatUpd_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        Set(TxtFeatUpdStatus, "Sizing feature-update leftovers…", StatusColors.Yellow);
        try
        {
            var scan = await Task.Run(FeatureUpdateCleanup.Scan);
            if (scan.Items.Count == 0)
            {
                Set(TxtFeatUpdStatus, "● Nothing to clean — no feature-update leftovers found.", StatusColors.Green);
                return;
            }

            UseLog(FeatUpdLogScroll, FeatUpdLog);
            Log($"▶ Feature-update leftovers scan (preview — nothing deleted) — {scan.TotalGb:F1} GB total");
            foreach (var i in scan.Items)
                Log($"  {i.Label} — {i.Gb:F1} GB{(i.Permanent ? "  ⚠ permanent" : "")}\n      {i.Path} · {i.Note}");
            Log($"━━━ {scan.TotalGb:F1} GB reclaimable — review, then click Clean ━━━");
            SaveLog("FeatUpdScan");

            Set(TxtFeatUpdStatus,
                $"● {scan.TotalGb:F1} GB across {scan.Items.Count} location(s) can be freed. See the log below."
                + (scan.AnyPermanent ? "  ⚠ includes Windows.old / staging (permanent)." : ""),
                StatusColors.Yellow);
        }
        catch (Exception ex) { Set(TxtFeatUpdStatus, "● " + ex.Message, StatusColors.Red); }
        finally { EndBusy(); }
    }

    private async void CleanFeatUpd_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        try
        {
            Set(TxtFeatUpdStatus, "Sizing…", StatusColors.Yellow);
            var scan = await Task.Run(FeatureUpdateCleanup.Scan);
            if (scan.Items.Count == 0)
            {
                Set(TxtFeatUpdStatus, "● Nothing to clean.", StatusColors.Green);
                return;
            }

            if (!TechGate.Verify(Window.GetWindow(this))) return;

            var breakdown = string.Join("\n", scan.Items.Select(i =>
                $"   • {i.Label} — {i.Gb:F1} GB{(i.Permanent ? "  (permanent)" : "")}"));
            string warn = scan.AnyPermanent
                ? "\n\n⚠ This includes Windows.old and/or upgrade staging. Removing them is PERMANENT and ends the " +
                  "~10-day option to roll back (\"go back\") to the previous Windows build."
                : "";
            if (!MessageWindow.Confirm("Feature-Update Leftovers",
                    $"Delete {scan.Items.Count} feature-update leftover location(s) — freeing {scan.TotalGb:F1} GB?",
                    $"Will remove:\n{breakdown}{warn}\n\nThis is what Windows Disk Cleanup's \"Previous Windows " +
                    "installation\" / \"Windows Update Cleanup\" options do. It cannot be undone. Continue?",
                    MessageKind.Warning, Window.GetWindow(this)))
                return;

            UseLog(FeatUpdLogScroll, FeatUpdLog);
            Set(TxtFeatUpdStatus, "Cleaning…", StatusColors.Yellow);
            var script = FeatureUpdateCleanup.BuildCleanScript(scan.Items);
            await RunPowerShell("pci_featupd_clean.ps1", script, "Feature-update cleanup", TxtFeatUpdStatus);

            // Re-measure so the tech sees what actually went (some items may need a reboot to finish).
            var after = await Task.Run(FeatureUpdateCleanup.Scan);
            long freed = scan.TotalBytes - after.TotalBytes;
            Log($"━━━ Freed ~{freed / 1073741824.0:F1} GB" +
                (after.Items.Count > 0 ? $"; {after.TotalGb:F1} GB remains (reboot may finish it) ━━━" : " ━━━"));
            Set(TxtFeatUpdStatus,
                after.Items.Count == 0
                    ? $"● Done — freed ~{freed / 1073741824.0:F1} GB."
                    : $"● Freed ~{freed / 1073741824.0:F1} GB; {after.TotalGb:F1} GB left (a reboot may be needed).",
                after.Items.Count == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        catch (Exception ex) { Set(TxtFeatUpdStatus, "● " + ex.Message, StatusColors.Red); }
        finally { EndBusy(); SaveLog("FeatUpdClean"); }
    }

    // ── WINDOWS UPDATE RESET ──────────────────────────────────────────────

    private const string WuResetScript = @"
$ErrorActionPreference = 'SilentlyContinue'
$services = 'wuauserv','bits','cryptsvc','msiserver','appidsvc'
Write-Output 'Stopping update services...'
foreach ($s in $services) { Stop-Service -Name $s -Force; Write-Output ""  stopped $s"" }
$stamp = Get-Date -Format 'yyyyMMdd_HHmmss'
foreach ($p in @(""$env:windir\SoftwareDistribution"", ""$env:windir\System32\catroot2"")) {
    if (Test-Path $p) {
        $leaf = Split-Path $p -Leaf
        Rename-Item -Path $p -NewName ""$leaf.bak_$stamp""
        if (Test-Path $p) { Write-Output ""  could not rename $leaf (in use)"" }
        else { Write-Output ""  cleared $leaf"" }
    }
}
Write-Output 'Starting update services...'
foreach ($s in $services) { Start-Service -Name $s; Write-Output ""  started $s"" }
Write-Output 'Windows Update components reset.'
";

    private async void WuReset_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!BeginServicing("Windows Update Reset")) return;   // stops/clears WU services — must not overlap Update All
        UseLog(WuLogScroll, WuLog);
        Set(TxtWuStatus, "Running…", StatusColors.Yellow);
        try
        {
            var code = await RunPowerShell("pci_wu_reset.ps1", WuResetScript,
                "Windows Update reset", TxtWuStatus);
            Set(TxtWuStatus, code == 0 ? "● Done — caches cleared" : "● Done (check log)",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        finally { EndServicing(); SaveLog("WUReset"); }
    }

    // ── ADVANCED CLEANUP ──────────────────────────────────────────────────

    private async void Cleanup_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!BeginServicing("Advanced Cleanup (WinSxS + WMI)")) return;
        UseLog(CleanupLogScroll, CleanupLog);
        Set(TxtCleanupStatus, "Running…", StatusColors.Yellow);
        try
        {
            await RunUtil("dism.exe", "/Online /Cleanup-Image /AnalyzeComponentStore", null,
                "DISM AnalyzeComponentStore", TxtCleanupStatus);
            await RunUtil("dism.exe", "/Online /Cleanup-Image /StartComponentCleanup", null,
                "DISM StartComponentCleanup", TxtCleanupStatus);
            await RunUtil("winmgmt.exe", "/verifyrepository", null,
                "WMI verifyrepository", TxtCleanupStatus);
            await RunUtil("winmgmt.exe", "/salvagerepository", null,
                "WMI salvagerepository", TxtCleanupStatus);
            Set(TxtCleanupStatus, "● Done", StatusColors.Green);
        }
        catch (Exception ex)
        {
            Set(TxtCleanupStatus, $"● Error: {ex.Message}", StatusColors.Red);
        }
        finally { EndServicing(); SaveLog("Cleanup"); }
    }

    // ── MAINTENANCE (moved from the old Actions tab) ──────────────────────

    private const string RestoreScript = @"
Enable-ComputerRestore -Drive 'C:\' -ErrorAction SilentlyContinue
New-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SystemRestore' `
    -Name 'SystemRestorePointCreationFrequency' -Value 0 -PropertyType DWord -Force -ErrorAction SilentlyContinue | Out-Null
Checkpoint-Computer -Description 'PartnerTool manual restore point' -RestorePointType 'MODIFY_SETTINGS'
Write-Output 'Restore point created.'
";


    private const string EmptyBinScript =
        "Clear-RecycleBin -Force -Confirm:$false -ErrorAction SilentlyContinue; Write-Output 'Recycle Bin emptied.'";

    private const string SpoolerScript =
        "Restart-Service -Name Spooler -Force; Write-Output 'Print spooler restarted.'";

    private const string ClearQueueScript = @"
Stop-Service -Name Spooler -Force
Remove-Item ""$env:windir\System32\spool\PRINTERS\*"" -Force -ErrorAction SilentlyContinue
Start-Service -Name Spooler
Write-Output 'Print queue cleared and spooler restarted.'
";

    // Lives in the System Restore Points card — writes its own status/log there and refreshes
    // the list so the new point shows up immediately.
    private async void RestorePoint_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        UseLog(RestoreLogScroll, RestoreLog);
        Set(TxtRestoreStatus, "Creating…", StatusColors.Yellow);
        try
        {
            var code = await RunPowerShell("pci_restore.ps1", RestoreScript, "Create restore point", TxtRestoreStatus);
            Set(TxtRestoreStatus, code == 0 ? "● Restore point created" : "● Done (check log)",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
            LoadRestorePoints();
        }
        finally { EndBusy(); SaveLog("RestorePoint"); }
    }

    // "Clear Temp Files" quick fix retired (0.17.66): it duplicated Clean Temp Files (All Users),
    // which covers more locations and has a Scan preview — one obvious way beats two similar ones.

    private async void EmptyBin_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;   // permanently deletes recycle-bin contents
        await RunQuickFix("pci_emptybin.ps1", EmptyBinScript, "Empty Recycle Bin");
    }

    private async void RestartSpooler_Click(object sender, RoutedEventArgs e)
        => await RunQuickFix("pci_spooler.ps1", SpoolerScript, "Restart print spooler");

    private async void ClearQueue_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;   // deletes pending print jobs
        await RunQuickFix("pci_clearqueue.ps1", ClearQueueScript, "Clear print queue");
    }

    // ── QUICK FIXES ───────────────────────────────────────────────────────

    private const string RestartExplorerScript = @"
Stop-Process -Name explorer -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 1
if (-not (Get-Process explorer -ErrorAction SilentlyContinue)) { Start-Process explorer.exe }
Write-Output 'Explorer restarted.'
";

    private const string RestartAudioScript = @"
Restart-Service -Name audiosrv -Force -ErrorAction SilentlyContinue
Write-Output 'Windows Audio service restarted.'
";

    private const string ReregStoreScript = @"
Get-AppxPackage -AllUsers Microsoft.WindowsStore | ForEach-Object {
    Add-AppxPackage -DisableDevelopmentMode -Register ""$($_.InstallLocation)\AppXManifest.xml"" -ErrorAction SilentlyContinue
}
Write-Output 'Microsoft Store re-registered.'
";

    private const string ClearIconCacheScript = @"
Stop-Process -Name explorer -Force -ErrorAction SilentlyContinue
Remove-Item ""$env:LocalAppData\IconCache.db"" -Force -ErrorAction SilentlyContinue
Remove-Item ""$env:LocalAppData\Microsoft\Windows\Explorer\iconcache*"" -Force -ErrorAction SilentlyContinue
Remove-Item ""$env:LocalAppData\Microsoft\Windows\Explorer\thumbcache*"" -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 1
if (-not (Get-Process explorer -ErrorAction SilentlyContinue)) { Start-Process explorer.exe }
Write-Output 'Icon and thumbnail caches cleared.'
";

    private async Task RunQuickFix(string fileName, string script, string startLog)
    {
        if (!BeginBusy()) return;
        UseLog(QuickFixLogScroll, QuickFixLog);
        Set(TxtQuickFixStatus, "Running…", StatusColors.Yellow);
        try
        {
            var code = await RunPowerShell(fileName, script, startLog, TxtQuickFixStatus);
            Set(TxtQuickFixStatus, code == 0 ? "● Done" : "● Done (check log)",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        finally { EndBusy(); SaveLog("QuickFix"); }
    }

    private async void RestartExplorer_Click(object sender, RoutedEventArgs e)
        => await RunQuickFix("pci_explorer.ps1", RestartExplorerScript, "Restart Explorer");

    private async void RestartAudio_Click(object sender, RoutedEventArgs e)
        => await RunQuickFix("pci_audio.ps1", RestartAudioScript, "Restart Audio");

    private async void ReregStore_Click(object sender, RoutedEventArgs e)
        => await RunQuickFix("pci_store.ps1", ReregStoreScript, "Re-register Microsoft Store");

    private async void ClearIconCache_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;   // kills Explorer + deletes cache files
        await RunQuickFix("pci_iconcache.ps1", ClearIconCacheScript, "Clear icon cache");
    }

    private void MemDiag_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        // mdsched is a GUI that prompts to restart now / on next boot — just launch it.
        // (Pinned to System32 — ShellExecute would search the working directory for a bare name.)
        try { Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemTool("mdsched.exe")) { UseShellExecute = true }); }
        catch (Exception ex)
        {
            MessageWindow.Show("Memory Diagnostic", "Couldn't launch", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }

    private async void SchedChkdsk_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;   // machine is unusable while it runs
        if (!MessageWindow.Confirm("Schedule CHKDSK", "Schedule a full disk check?",
                "This schedules chkdsk C: /f /r to run at the next restart. It can take a long time " +
                "and the PC is unusable while it runs. Continue?", MessageKind.Warning, Window.GetWindow(this)))
            return;
        if (!BeginBusy()) return;
        UseLog(ChkdskLogScroll, ChkdskLog);
        Set(TxtChkdskStatus, "Scheduling…", StatusColors.Yellow);
        try
        {
            // Pipe Y to answer the "schedule at next boot?" prompt.
            var code = await RunUtil("cmd.exe", "/c echo Y| chkdsk C: /f /r", null,
                "Schedule CHKDSK C: /f /r at next boot", TxtChkdskStatus);
            Set(TxtChkdskStatus, "● Scheduled — runs at next restart", StatusColors.Green);
        }
        finally { EndBusy(); SaveLog("Chkdsk"); }
    }

    // ── REPORTS ───────────────────────────────────────────────────────────

    private async void BatteryReport_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        _logText = null;   // Reports just open a browser — no inline log for this card
        Set(TxtReportStatus, "Generating…", StatusColors.Yellow);
        try
        {
            var path = Path.Combine(Path.GetTempPath(), "PartnerTool-battery-report.html");
            await RunUtil("powercfg.exe", $"/batteryreport /output \"{path}\"", null, "Battery report", TxtReportStatus);
            if (File.Exists(path))
            {
                Set(TxtReportStatus, "● Opened in your browser", StatusColors.Green);
                Process.Start(new ProcessStartInfo(path) { UseShellExecute = true });
            }
            else
                Set(TxtReportStatus, "● No battery detected (desktop?)", StatusColors.Yellow);
        }
        finally { EndBusy(); SaveLog("Report"); }
    }

    private async void GpReport_Click(object sender, RoutedEventArgs e)
    {
        if (!BeginBusy()) return;
        _logText = null;   // Reports just open a browser — no inline log for this card
        Set(TxtReportStatus, "Generating…", StatusColors.Yellow);
        try
        {
            var path = Path.Combine(Path.GetTempPath(), "PartnerTool-gpreport.html");
            await RunUtil("gpresult.exe", $"/h \"{path}\" /f", null, "Group Policy report", TxtReportStatus);
            if (File.Exists(path))
            {
                Set(TxtReportStatus, "● Opened in your browser", StatusColors.Green);
                Process.Start(new ProcessStartInfo(path) { UseShellExecute = true });
            }
            else
                Set(TxtReportStatus, "● Could not generate the report", StatusColors.Yellow);
        }
        finally { EndBusy(); SaveLog("Report"); }
    }

    // ── SYSTEM RESTORE POINTS ─────────────────────────────────────────────

    private void LoadRestorePoints()
    {
        var points = RestorePointsInfo.Collect();
        IcRestorePoints.ItemsSource = points;
        TxtNoRestore.Visibility = points.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
    }

    private void RefreshRestore_Click(object sender, RoutedEventArgs e) => LoadRestorePoints();

    private void OpenRestore_Click(object sender, RoutedEventArgs e)
    {
        try { RestorePointsInfo.OpenWizard(); }
        catch (Exception ex)
        {
            MessageWindow.Show("System Restore", "Couldn't open System Restore", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }

    // ── COLLECT DIAGNOSTICS ───────────────────────────────────────────────

    private async void CollectDiag_Click(object sender, RoutedEventArgs e)
    {
        var dlg = new Microsoft.Win32.SaveFileDialog
        {
            FileName   = $"Diagnostics_{Environment.MachineName}_{DateTime.Now:yyyyMMdd_HHmmss}",
            DefaultExt = ".zip",
            Filter     = "Zip archive (*.zip)|*.zip",
        };
        if (dlg.ShowDialog() != true) return;
        if (!BeginBusy()) return;
        Set(TxtCollectStatus, "Collecting…", StatusColors.Yellow);
        try
        {
            ActivityLog.Action("Repair", $"Create diagnostics bundle → {dlg.FileName}");
            var snap = await SystemSnapshot.CaptureAsync();
            await DiagnosticsBundle.CreateAsync(dlg.FileName, snap);
            Set(TxtCollectStatus, "● Saved bundle", StatusColors.Green);
        }
        catch (Exception ex) { Set(TxtCollectStatus, $"● {ex.Message}", StatusColors.Red); }
        finally { EndBusy(); }
    }

}
```
