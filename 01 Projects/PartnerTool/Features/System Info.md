---
project: PartnerTool
tags: [partnertool, feature]
---

# System Info (page)

The landing page — full machine picture, painted from the startup snapshot.

- **LIVE STATS** tiles (CPU/RAM/Disk/Net sparklines, 1 Hz off-thread sampling via `LivePerf`); clicking a tile opens the Task-Manager-style **PerformanceWindow** (per-core CPU grid from `PerfDetails`, advanced RAM/disk/net stats).
- CPU package power (from LibreHardwareMonitor) only shows when the sensor reports ≥ 1 W — hidden as meaningless "0 W" noise on machines where the driver is HVCI-blocked (0.17.56).
- Cards: Identity (incl. Azure AD/domain join from `dsregcmd`), Hardware (disks + SMART summary, RAM slots, GPU, BIOS/TPM), OS, Performance (page file + proxy rows), Power Status (power mode with inline `(change)` → PowerPlanWindow; fast startup; hibernation; sleep timers; last wake), Battery (incl. live voltage from `root\wmi BatteryStatus`), Monitors, Printers, Local Accounts, **Network** (the default-route adapter, labelled "… (Default)"), Warranty (manual lookup button — auto-pull deferred, needs Dell TechDirect key).
- **Storage card (0.19.4):** each disk shows the inline SMART **summary** (temp/wear/hours/errors from the Storage reliability counters); a **SMART details** button opens `SmartWindow` — the full CrystalDiskInfo-style attribute table (ID/current/worst/threshold/raw, failing rows + predict-failure flagged red) from the previously-unused `SmartAttributes` collector. SATA/ATA only; the window explains NVMe doesn't expose the classic table.
- **Monitors + Memory are tables (0.19.4):** Monitors = Make / Model / Size / Year / Serial; Memory = Slot / Size / Speed / Maker. Aligned via `Grid.IsSharedSizeScope` + `SharedSizeGroup` columns (header shares widths with rows); display props live on `DisplayInfo` / `MemoryModule`.
- **Power settings shortcuts (0.19.4):** `(change)` links by Fast startup / Hibernation open `control.exe /name Microsoft.PowerOptions /page pageGlobalSettings`; Sleep after opens `ms-settings:powersleep`.
- **Auto-refresh:** snapshot data refreshes every **45 seconds** while visible ("Refreshed at HH:mm:ss · Updates every 45 seconds"). No manual Refresh button (0.17.55) — the timer is always-on at Normal priority with visibility-gated work, so it can't be left un-started (see [[WPF Single-File Quirks]] §2c).
- **Power buttons:** Restart / Shut Down / Sign Out (tech-gated + confirmed), Lock (ungated) — all logged.
