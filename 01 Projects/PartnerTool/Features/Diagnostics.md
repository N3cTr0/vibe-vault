---
project: PartnerTool
tags: [partnertool, feature]
---

# Diagnostics (page)

Card order (0.17.66): **Reliability → Crash History → Health History → Device Problems → Top Processes → Power & Restart → Boot → Recent Errors** — Crash History sits directly under Reliability because the reliability caption points to it. No top Refresh button — the page refreshes on each open.

- **Reliability:** stability index (0–10) from `Win32_ReliabilityStabilityMetrics`. **Important context** (see [[Lessons Learned]]): Windows drops this for app installs, updates and driver changes — *not just crashes* — so a busy but stable machine can read ~1.5/10. The page shows an explanatory caption plus the **7-day best** (`ReliabilityInfo.RecentPeak`) so it isn't misread. "Load detailed history" opens the slow `Win32_ReliabilityRecords` (~a minute) in a **popup**.
- **Health History:** per-day trend from `SnapshotHistory` (disk free, RAM, batt wear, stability). Shows the last **7 days**; "More…" opens the full trend in a popup.
- **Device problems:** `Win32_PnPEntity` ConfigManagerErrorCode mapped to friendly text.
- **Crash history:** minidump count/latest + MEMORY.DMP presence; BSOD bugchecks and application crashes from the event log; boot performance (last boot time + top contributors).
- **Recent errors:** Critical/Error events from System+Application (last 10 days), "More…" opens the full list.
- **Power & restart history**.
- **Live processes:** top by usage; **End Process** is tech-gated + confirmed, blocks the BSOD-critical set **and svchost** (see [[Conventions & Security Model]]).

Debugging note: app's own crashes land in `C:\PCI\Logs\PartnerTool_errors.log` and this page's Application Crashes list — which is how the shutdown-telemetry bug was caught ([[WPF Single-File Quirks]]).
