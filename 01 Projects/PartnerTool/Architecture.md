---
project: PartnerTool
tags: [partnertool]
---

# Architecture

## Startup flow

1. `App.OnStartup` — hardens `C:\PCI` ACLs (`SecureDirectory.EnsureHardened`), wires crash handlers, loads `settings.json`.
2. `MainWindow.StartupAsync` — behind the splash: `SystemSnapshot.CaptureAsync()` runs **every collector in parallel**; pages `Render(snap)` from it; live-perf tiles are pre-warmed.
3. Pages are **singletons** cached in `MainWindow`, swapped via `PageHost.Content`. Lazy loads hang off `Loaded` / `IsVisibleChanged`.
4. `MainWindow.OnClosed` → `TerminateProcess(GetCurrentProcess(), 0)` — deliberate hard-terminate (skips the WPF/native exit teardown that faulted on close; `Environment.Exit(0)` was the earlier attempt and still faulted — 0.17.59); see [[WPF Single-File Quirks]].

## Key classes

| Class | Role |
|---|---|
| `SystemSnapshot` | One point-in-time capture of everything the info tabs show |
| `ProcessRunner` | Bounded CLI runner. `RunAsync` auto-logs to ActivityLog + parses DISM % progress; `RunCaptureAsync` is quiet (read-only queries). `ResolveSystemExe` pins bare names to System32 (anti-planting) |
| `ActivityLog` | Append-only audit trail → `C:\PCI\Logs\PartnerTool_activity.log` (see [[Tech Gate & Activity Log]]) |
| `TechGate` / `TechGateWindow` | Once-per-session verification for destructive actions (masked PasswordBox) |
| `LivePerf` / `PerfDetails` | 1 Hz live sampling + Task-Manager-style PerformanceWindow (per-core CPU) |
| `MftVolume` | Raw NTFS $MFT reader for [[MFT Fast Scan]] |
| `FullReport` / `ReportBuilder` / `DiagnosticsBundle` | Collect-everything report (HTML + text) — every new collector must be added here (see [[Conventions & Security Model]]) |
| `SoftwareInventory` | Shared installed-software scan (Software page + report) |
| `SettingsStore` | `settings.json` next to the exe, edited via the **Settings popup** (sidebar, back in 0.17.68): `LogRetentionDays` (default 30), the `EnableSensors` kill-switch (LibreHardwareMonitor driver), and `ShowPreviewFeatures` (Health Check tab toggle, default `true` since 0.19.4); AssetTag/Notes removed 0.17.69 |
| `ServicingLock` | App-wide gate so CBS/TrustedInstaller-heavy ops (Full Repair, Advanced Cleanup, CHKDSK, WU Reset, Update All) can't overlap across pages — prevents DISM 0x800F0915-style contention failures (0.17.68) |
| `LogRetention` | Startup prune of `C:\PCI\Logs` older than `LogRetentionDays` — the folder is ACL-hardened, so only the elevated app can tidy it; pruning is activity-logged |
| `HealthCheck` / `ShortcutScan` | [[Health Check]] engine — parallel read-only reuse of existing collectors → scored findings; ShortcutScan finds dead `.lnk` and recycles them (SHFileOperation FOF_ALLOWUNDO) |

## Collector modules (one class ≈ one concern)

`SystemInfo`, `PerfSnapshot`, `SecuritySnapshot`, `SecurityAudit` (hardening scorecard + clickable fixes), `HardwareInfo` (disks/SMART, RAM, GPU, TPM, battery wear), `NetworkInfo` (primary + all adapters), `WifiInfo`, `DiagnosticsInfo` (events, device problems, dumps), `ReliabilityInfo`, `UpdateHistoryInfo`, `DisplaysInfo`, `PrintersInfo` (list + protected-queue guard + removal),
`SharesInfo` (`Win32_Share`, admin shares flagged by the `0x80000000` type bit), `WuPolicyInfo` (update
source + reset-to-online), `DevModeInfo` (Developer Mode toggle), `BootPerfInfo`, `CrashInfo`,
`RestorePointsInfo`, `ProsentryInfo` (managed security stack + Intune), `AccountsInfo`, `SystemExtras` (page file, proxy, power plan), `AzureAdInfo` (dsregcmd), `PowerStatusInfo` (powercfg), `BatteryLive`, `StartupInfo`, `ServicesInfo`, `ProcessInfo`, `VendorUpdatesInfo` (Dell/Lenovo/HP), `PendingUpdatesInfo`, `OutdatedAppsInfo` (winget), `StoreUpdatesInfo` (pending Store/UWP updates, read-only), `OfficeLanguages`, `InstallerCleanup`, `TempCleaner`, `ProxyRepair`, `DiskUsageInfo`.

## Load timings (`LoadTimer`, 0.24.15)

Every page load, plus the startup snapshot and the Health Check scan, is timed. Wrap a collector
with `t.Time("name", Collect)` — a drop-in for `Task.Run` — and call `t.Done()`; one line goes to the
activity log under the **`perf`** category with the wall time and every part, **slowest first**:

```
[perf] Security load 126 ms  (defender 122, prosentry 72, audit 35, bitlocker 12)
[perf] Manage load 8441 ms  (tasks 8441)
```

The parts sum to more than the wall time when they ran in parallel, and to roughly it when they ran
one after another — which is itself worth seeing (Health Check is deliberately sequential so its
progress bar can be honest, so its parts sum to what the tech waits for). `FirstAndSlowOnly()` is for
loads that also run on a timer or on every page re-open — they log the first one, then only loads
over 2 s. The diagnostics bundle already collects the activity log, so a slow machine in the field
reports its own hot spots.

**Known hot spots (dev VM, 2026-07-31):** Manage ▸ scheduled tasks 7–8 s (`schtasks /query /fo CSV /v`
— the `/v` is the cost *and* the source of the Last Run Time / Last Result columns), Manage ▸ drivers
1.3 s, Updates ▸ Windows 8–15 s and Store 6.6–7.2 s (both network-bound). Startup snapshot 1122 ms
wall across 16 parallel collectors. Everything else is sub-second.

Full source: [[_Code Index]].
