---
project: PartnerTool
tags: [partnertool, feature]
---

# Health Check (page)

Added 0.18.0. The friendly one-scan view — a curated front-end over checks the tool **already**
does, born from researching IObit Advanced SystemCare and deliberately leaving out its junk (see the
verdict at the bottom). Sidebar button sits **above Settings** (kept out of the main alphabetical nav
while we iterate).

**Visible by default (0.19.4).** It was hidden behind a preview toggle while being built (0.18.2); now
that it's matured the sidebar tab shows out of the box (`ShowPreviewFeatures` default **true**). The
Settings toggle — relabelled **"Show the Health Check tab"** — can still hide it.
`MainWindow.ApplyPreviewVisibility` reacts live to `SettingsStore.Changed` and falls back to System
Info if you hide the tab while it's open.

## How it works

- A big **score ring** (200px) is the button — click it to scan/rescan. `HealthCheck.RunAsync()`
  runs all the existing read-only collectors **in parallel** and returns a `HealthReport`: a 0–100
  score (100 minus each finding's deduction, clamped) and a list of `HealthFinding`s.
- Ring colour: ≥85 green, ≥60 yellow, else red.
- Each finding is one of:
  - **Fixable** (has a `Fixer`) → gets a **checkbox** (default ticked) with the fix verb; run by the
    single **Fix Selected** button. The batch is **tech-gated once** if any selected fix deletes
    files (`NeedsGate`), then rescans automatically so fixed items go green.
  - **Advisory** (has a `NavTag`) → an **Open →** button that jumps to the relevant tab
    (`MainWindow.NavigateTo`, made `internal` for this).
  - **Good/Info** → shown as a green row so the tech sees what was checked and passed.

## What it checks (all reused, read-only)

| Category | Source | Fixable? |
|---|---|---|
| Temp & cache files | `TempCleaner.Scan` | ✅ Clean (gated) |
| Windows Installer orphans | `InstallerCleanup.Scan` | ✅ Clean (gated) |
| Broken shortcuts | **`ShortcutScan`** (new) | ✅ Recycle (reversible, not gated) |
| Disk free space + SMART | `HardwareInfo` | Advisory → Disk Usage / System Info |
| Pending updates | `PendingUpdatesInfo` + `OutdatedAppsInfo` | Advisory → Updates |
| Security hardening + antivirus | `SecurityAudit` + `DefenderInfo` | Advisory → Security |
| Crashes/BSODs (14d) + device problems | `CrashInfo` + `DiagnosticsInfo` | Advisory → Diagnostics |
| Uptime / reboot-pending | `PerfSnapshot` + registry check | Advisory |
| Startup load | `StartupInfo` | Advisory → Software |
| Dell SupportAssist snapshots + VSS (Dell only) | **`DellRemediation`** (new, 0.19.0) | Snapshots advisory · VSS ✅ Cap (gated) |

**`DellRemediation`** (0.19.0, Dell hardware only) sizes `C:\ProgramData\Dell\SARemediation` and reads
VSS shadow-storage limits from WMI (`Win32_ShadowStorage`, matched to C: via `Win32_Volume` DeviceID —
no wmic, per [[Conventions & Security Model]]). The snapshot folder is **reported, never deleted**
(Dell says hand-deleting breaks OS Recovery — the tech turns System Repair off in SupportAssist). The
one **fixable** part is **unbounded shadow storage** (Dell KB 000129138): the Cap fix runs
`vssadmin resize shadowstorage /For=C: /On=C: /MaxSize=5%` — tech-gated + confirmed (it discards
existing restore points). Cap % is `DellRemediation.VssMaxPercent` (5% house default; Dell's KB says 2%). Also surfaced as a **Repair-page card** ([[Repair]]) so it's reachable
without the in-progress toggle. See [[Lessons Learned]] for the Dell space research.

**`ShortcutScan`** is the one new collector: finds `.lnk` on Desktop + Start Menu whose target no
longer exists (conservative — only absolute paths on ready fixed drives, never UNC/removable), and
recycles them via `SHFileOperation` with `FOF_ALLOWUNDO` (goes to Recycle Bin, reversible).

## Why we did NOT copy the rest of ASC

Research verdict (see [[Lessons Learned]]): Microsoft **doesn't support registry cleaners**,
Malwarebytes calls them "digital snake oil", and ASC itself is flagged PUP by some AVs / bundles /
upsells. So Health Check has **no registry clean/defrag, no privacy/browser-history wiping (an MSP
liability + we deliberately exclude browser data from the temp cleaner), and no placebo "internet
boost / system optimization"**. That makes it *more* defensible in front of a client than ASC, not
less — every number is a real check and every fix is one we already trust and log.
