---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\HealthCheck.cs
---

# PartnerTool\HealthCheck.cs

```csharp
using System.Windows;
using Microsoft.Win32;

namespace PartnerTool;

public enum HealthSeverity { Good, Warn, Bad }

/// <summary>
/// One row in the Health Check results. A finding is either FIXABLE in place (has a
/// <see cref="Fixer"/> → gets a checkbox and is run by "Fix Selected") or ADVISORY (has a
/// <see cref="NavTag"/> → shows an "Open" button that jumps to the relevant tab), or purely
/// informational. Deliberately: everything here reuses PartnerTool's existing, trustworthy checks —
/// no registry "cleaning", no privacy wiping, no placebo "optimization".
/// </summary>
public class HealthFinding
{
    public required string Category { get; init; }
    public required string Title    { get; init; }
    public string Detail            { get; set; } = "";
    public HealthSeverity Severity  { get; init; }
    public int Deduction            { get; init; }   // score points off (0 for Good/Info)

    /// <summary>In-place fix (logs via the callback, returns a one-line result). Null = not fixable.</summary>
    public Func<Action<string>, Task<string>>? Fixer { get; init; }
    public string FixLabel  { get; init; } = "Fix";
    public bool   NeedsGate { get; init; }           // tech-gate the batch if any selected finding sets this
    public bool   Selected  { get; set; }            // checkbox (two-way); defaults on for fixables

    /// <summary>Advisory: nav tag of the tab that handles this (MainWindow nav), or null.</summary>
    public string? NavTag { get; init; }

    public bool IsFixable => Fixer != null;
    public Visibility CheckVis => IsFixable ? Visibility.Visible : Visibility.Collapsed;
    public Visibility NavVis   => (!IsFixable && NavTag != null) ? Visibility.Visible : Visibility.Collapsed;
}

public class HealthReport
{
    public List<HealthFinding> Findings { get; } = new();
    public int Score { get; set; }
    public int ProblemCount => Findings.Count(f => f.Severity != HealthSeverity.Good);
    public int FixableCount => Findings.Count(f => f.IsFixable);
}

/// <summary>
/// One-scan machine health check: runs PartnerTool's existing read-only collectors in parallel and
/// turns them into a scored list of findings, some fixable in place. It's the friendly "SCAN" view
/// — a curated front-end over checks we already trust, NOT an IObit-style optimizer.
/// </summary>
public static class HealthCheck
{
    /// <param name="progress">Reports (percent 0–100, current step) as the scan runs. Invoked on the
    /// caller's context (we don't ConfigureAwait(false)) so the UI can update directly.</param>
    public static async Task<HealthReport> RunAsync(Action<int, string>? progress = null)
    {
        void P(int pct, string msg) => progress?.Invoke(pct, msg);
        var r = new HealthReport();

        // Sequential stages so we can report honest progress + a "what it's doing now" message.
        P(5,  "Scanning temp & cache files…");
        var temp = await Task.Run(() => TempCleaner.Scan());

        P(18, "Checking the Windows Installer cache…");
        var inst = await Task.Run(InstallerCleanup.Scan);

        P(30, "Scanning Desktop & Start Menu shortcuts…");
        var broken = await Task.Run(ShortcutScan.Scan);

        P(42, "Reading disk space & drive health…");
        var hw = await Task.Run(HardwareInfo.Collect);

        P(55, "Checking for Windows & app updates…");
        int winCount = (await Task.Run(PendingUpdatesInfo.Collect)).Count;
        int appCount = (await OutdatedAppsInfo.CollectAsync()).Count;

        P(72, "Checking security & antivirus…");
        var audit = await Task.Run(SecurityAudit.Collect);
        var def   = await Task.Run(DefenderInfo.Collect);

        P(85, "Checking crash history & devices…");
        var crash = await Task.Run(CrashInfo.Collect);
        var diag  = await Task.Run(DiagnosticsInfo.Collect);

        P(90, "Checking Dell SupportAssist & shadow storage…");
        var dell = await Task.Run(DellRemediation.Collect);

        P(94, "Checking uptime & startup programs…");
        var perf       = await Task.Run(PerfSnapshot.Collect);
        var startupAll = await Task.Run(StartupInfo.Collect);
        bool rebootPending = await Task.Run(RebootPending);

        P(99, "Scoring…");

        // ── Junk: temp files ─────────────────────────────────
        double tempGb = temp.TotalBytes / 1073741824.0;
        if (tempGb >= 1.0)
            r.Findings.Add(new HealthFinding
            {
                Category = "Junk", Title = "Temp & cache files", Severity = HealthSeverity.Warn, Deduction = 4,
                Detail = $"{temp.TotalBytes / 1048576.0:N0} MB across {temp.ProfilesProcessed} profile(s) can be cleaned.",
                FixLabel = "Clean", NeedsGate = true, Selected = true,
                Fixer = log => Task.Run(() => { var res = TempCleaner.Clean(null, log); return res.Summary; }),
            });
        else
            r.Findings.Add(Good("Junk", "Temp & cache files", $"{temp.TotalBytes / 1048576.0:N0} MB — nothing significant to clean."));

        // ── Junk: orphaned installer files ───────────────────
        if (inst.KeepSetValid && inst.Orphans.Count > 0 && inst.OrphanGb >= 0.5)
            r.Findings.Add(new HealthFinding
            {
                Category = "Junk", Title = "Windows Installer orphans", Severity = HealthSeverity.Warn, Deduction = 4,
                Detail = $"{inst.Orphans.Count} orphaned .msi/.msp file(s) — {inst.OrphanGb:F1} GB reclaimable.",
                FixLabel = "Clean", NeedsGate = true, Selected = true,
                Fixer = log => Task.Run(() =>
                {
                    var (deleted, freed, failed) = InstallerCleanup.DeleteOrphans(inst.Orphans, log);
                    return $"Removed {deleted} file(s), freed {freed / 1073741824.0:F1} GB"
                         + (failed > 0 ? $"; {failed} in use/skipped." : ".");
                }),
            });
        else
            r.Findings.Add(Good("Junk", "Windows Installer orphans", "No significant orphaned installer files."));

        // ── Broken shortcuts ─────────────────────────────────
        if (broken.Count > 0)
            r.Findings.Add(new HealthFinding
            {
                Category = "Shortcuts", Title = "Broken shortcuts", Severity = HealthSeverity.Warn, Deduction = 3,
                Detail = $"{broken.Count} shortcut(s) point to files that no longer exist (Desktop / Start Menu).",
                FixLabel = "Recycle", NeedsGate = false, Selected = true,
                Fixer = log => Task.Run(() =>
                {
                    int n = ShortcutScan.Recycle(broken.Select(b => b.LnkPath), log);
                    return $"Sent {n} broken shortcut(s) to the Recycle Bin.";
                }),
            });
        else
            r.Findings.Add(Good("Shortcuts", "Broken shortcuts", "No broken shortcuts found."));

        // ── Disk: free space (system volume) ─────────────────
        var sys = hw.Volumes.FirstOrDefault(v => v.Letter.StartsWith("C", StringComparison.OrdinalIgnoreCase))
                  ?? hw.Volumes.FirstOrDefault();
        if (sys != null)
        {
            double freePct = 100 - sys.UsedPct;
            if (freePct < 10)
                r.Findings.Add(Advisory("Disk", $"Low disk space on {sys.Letter}", HealthSeverity.Bad, 12,
                    $"Only {sys.FreeGb:F0} GB free ({freePct:F0}%). Windows may become unstable below ~10%.", "DiskUsage"));
            else if (freePct < 15)
                r.Findings.Add(Advisory("Disk", $"Disk space low on {sys.Letter}", HealthSeverity.Warn, 6,
                    $"{sys.FreeGb:F0} GB free ({freePct:F0}%).", "DiskUsage"));
            else
                r.Findings.Add(Good("Disk", $"Disk space on {sys.Letter}", $"{sys.FreeGb:F0} GB free ({freePct:F0}%)."));
        }

        // ── Disk: SMART health ───────────────────────────────
        var badDisk = hw.Disks.FirstOrDefault(d => !IsHealthy(d.Health));
        if (badDisk != null)
            r.Findings.Add(Advisory("Disk", "Drive health warning", HealthSeverity.Bad, 20,
                $"{badDisk.Model}: SMART status “{badDisk.Health}”. Back up and plan a replacement.", "SysInfo"));
        else if (hw.Disks.Count > 0)
            r.Findings.Add(Good("Disk", "Drive health (SMART)", "All drives report healthy."));

        // ── Dell: SupportAssist snapshots + VSS shadow storage ──
        // Only ever shown on Dell hardware. The snapshot folder is REPORTED, never deleted (Dell says
        // hand-deleting breaks OS Recovery — System Repair must be turned off in SupportAssist).
        // Unbounded shadow storage IS fixable, and is Dell's own documented remedy (KB 000129138).
        if (dell.IsDell && dell.FolderExists)
        {
            if (dell.VeryBad)
                r.Findings.Add(Advisory("Dell", "SupportAssist snapshots are huge", HealthSeverity.Bad, 8,
                    $"{DellRemediation.RootPath} is using {dell.TotalGb:F1} GB. Turn System Repair off in " +
                    "SupportAssist to purge it — don't delete the folder by hand.", null));
            else if (dell.Bloated)
                r.Findings.Add(Advisory("Dell", "SupportAssist snapshots are large", HealthSeverity.Warn, 4,
                    $"{DellRemediation.RootPath} is using {dell.TotalGb:F1} GB (Dell reserves 15 GB by default).", null));
            else
                r.Findings.Add(Good("Dell", "SupportAssist snapshots", $"{dell.TotalGb:F1} GB — within Dell's normal range."));
        }

        if (dell.IsDell && dell.VssUnbounded)
            r.Findings.Add(new HealthFinding
            {
                Category = "Dell", Title = "VSS shadow storage is unbounded", Severity = HealthSeverity.Warn, Deduction = 5,
                Detail = $"Shadow copies on C: have no size limit (currently {dell.VssUsedGb:F1} GB) — Dell KB 000129138. " +
                         $"Capping at {DellRemediation.VssMaxPercent}% stops the runaway growth, but discards existing " +
                         "System Restore points. Tick to apply.",
                FixLabel = $"Cap at {DellRemediation.VssMaxPercent}%", NeedsGate = true, Selected = false,
                Fixer = log => DellRemediation.BoundVssAsync(log),
            });
        else if (dell.IsDell && dell.VssConfigured)
            r.Findings.Add(Good("Dell", "VSS shadow storage", $"Capped at {dell.VssMaxText} on C:."));
        else if (dell.IsDell && dell.VssQueried)
            r.Findings.Add(Good("Dell", "VSS shadow storage", $"{dell.VssMaxText} on C: — nothing to bound."));

        // ── Updates ──────────────────────────────────────────
        int updTotal = winCount + appCount;
        if (updTotal > 0)
            r.Findings.Add(Advisory("Updates", "Updates available", HealthSeverity.Warn, 5,
                $"{winCount} Windows update(s), {appCount} app update(s) pending.", "Updates"));
        else
            r.Findings.Add(Good("Updates", "Updates", "No pending Windows or app updates found."));

        // ── Security: hardening scorecard ────────────────────
        int bad  = audit.Count(a => a.Level == AuditLevel.Bad);
        int warn = audit.Count(a => a.Level == AuditLevel.Warn);
        if (bad > 0)
            r.Findings.Add(Advisory("Security", "Security hardening issues", HealthSeverity.Bad, 12,
                $"{bad} high-priority + {warn} minor item(s) flagged on the hardening scorecard.", "Security"));
        else if (warn > 0)
            r.Findings.Add(Advisory("Security", "Minor hardening items", HealthSeverity.Warn, 5,
                $"{warn} minor item(s) flagged on the hardening scorecard.", "Security"));
        else
            r.Findings.Add(Good("Security", "Security hardening", "No hardening issues flagged."));

        // ── Security: antivirus (only when Defender is the active AV) ──
        if (def.Available && def.AntivirusEnabled)
        {
            if (!def.RealTimeProtection)
                r.Findings.Add(Advisory("Security", "Real-time protection off", HealthSeverity.Bad, 20,
                    "Microsoft Defender real-time protection is disabled.", "Security"));
            else if (def.SignatureUpdated is { } sig && (DateTime.Now - sig).TotalDays > 7)
                r.Findings.Add(Advisory("Security", "Antivirus signatures out of date", HealthSeverity.Warn, 6,
                    $"Defender signatures last updated {(int)(DateTime.Now - sig).TotalDays} days ago.", "Updates"));
            else
                r.Findings.Add(Good("Security", "Antivirus", "Defender active with recent signatures."));
        }

        // ── Stability: crashes (last 14 days) ────────────────
        var cutoff = DateTime.Now.AddDays(-14);
        int bsods   = crash.Bsods.Count(b => b.Time >= cutoff);
        int appErrs = crash.Crashes.Count(c => c.Time >= cutoff);
        if (bsods > 0)
            r.Findings.Add(Advisory("Stability", "Blue screens recently", HealthSeverity.Bad, 15,
                $"{bsods} blue-screen crash(es) in the last 14 days.", "Diag"));
        else if (appErrs >= 3)
            r.Findings.Add(Advisory("Stability", "Frequent app crashes", HealthSeverity.Warn, 5,
                $"{appErrs} application crash(es) in the last 14 days.", "Diag"));
        else
            r.Findings.Add(Good("Stability", "Crash history", "No blue screens in the last 14 days."));

        // ── Stability: device problems ───────────────────────
        int devs = diag.Devices.Count;
        if (devs > 0)
            r.Findings.Add(Advisory("Stability", "Device problems", HealthSeverity.Warn, 6,
                $"{devs} device(s) reporting a driver/hardware problem.", "Diag"));
        else
            r.Findings.Add(Good("Stability", "Device manager", "No devices reporting problems."));

        // ── Maintenance: uptime / reboot pending ─────────────
        double upDays = perf.BootKnown ? (DateTime.Now - perf.LastBoot).TotalDays : 0;
        if (rebootPending)
            r.Findings.Add(Advisory("Maintenance", "Restart required", HealthSeverity.Warn, 3,
                "Windows has a pending reboot (updates/installs won't finish until you restart).", null));
        else if (upDays > 14)
            r.Findings.Add(Advisory("Maintenance", "Long uptime", HealthSeverity.Warn, 3,
                $"Up for {upDays:F0} days — a restart clears leaks and applies staged updates.", null));
        else
            r.Findings.Add(Good("Maintenance", "Uptime", perf.BootKnown ? $"Up for {upDays:F0} day(s)." : "Uptime unavailable."));

        // ── Startup load (informational) ─────────────────────
        int enabled = startupAll.Count(s => s.Enabled);
        if (enabled > 15)
            r.Findings.Add(Advisory("Startup", "Many startup programs", HealthSeverity.Warn, 3,
                $"{enabled} programs run at sign-in — review to speed up boot.", "Software"));
        else
            r.Findings.Add(Good("Startup", "Startup programs", $"{enabled} program(s) run at sign-in."));

        r.Score = Math.Clamp(100 - r.Findings.Sum(f => f.Deduction), 0, 100);
        return r;
    }

    private static HealthFinding Good(string cat, string title, string detail) =>
        new() { Category = cat, Title = title, Detail = detail, Severity = HealthSeverity.Good, Deduction = 0 };

    private static HealthFinding Advisory(string cat, string title, HealthSeverity sev, int deduction, string detail, string? navTag) =>
        new() { Category = cat, Title = title, Detail = detail, Severity = sev, Deduction = deduction, NavTag = navTag };

    private static bool IsHealthy(string h) =>
        string.IsNullOrWhiteSpace(h) ||
        h.Contains("ok", StringComparison.OrdinalIgnoreCase) ||
        h.Contains("healthy", StringComparison.OrdinalIgnoreCase);

    /// <summary>Standard Windows "reboot pending" registry markers (CBS, Windows Update, pending renames).</summary>
    private static bool RebootPending()
    {
        try
        {
            using (var cbs = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"))
                if (cbs != null) return true;
            using (var wu = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired"))
                if (wu != null) return true;
            using (var sm = Registry.LocalMachine.OpenSubKey(@"SYSTEM\CurrentControlSet\Control\Session Manager"))
                if (sm?.GetValue("PendingFileRenameOperations") != null) return true;
        }
        catch { }
        return false;
    }
}
```
