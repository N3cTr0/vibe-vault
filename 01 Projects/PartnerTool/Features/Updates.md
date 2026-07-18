---
project: PartnerTool
tags: [partnertool, feature]
---

# Updates (page)

Layout (redesigned 0.17.56): **one card per update source**, each merging its scan result *with* its launcher, instead of a separate "available updates" summary block + disconnected launcher cards:

- **Windows Update** — "Open" (ms-settings:windowsupdate) + the pending-updates list (`PendingUpdatesInfo`). Sizes are adaptive KB/MB/GB; anything > 4 GB shows "—" because WUA's MaxDownloadSize on UUP cumulative updates is the whole payload range (~90 GB), not the real delta download (PSWindowsUpdate prints the same raw value in its table).
- **App Updates (winget)** — the outdated-apps list (`OutdatedAppsInfo`; no `--include-unknown` — "Unknown → x" rows are noise); upgraded by Update All.
- **Manufacturer** (dynamic: Dell Command Update / Lenovo System Update / HP Image Assistant) — "Open" + Installed status + the driver/BIOS scan (`VendorUpdatesInfo`, mirrors the production NinjaOne HUS script in `reference/HUS_MultiVendorUpdate_v1.14.1.ps1`; auto-scan never auto-installs the vendor tool). **VM-aware** (0.17.58): `ManufacturerTools.Get(maker, model)` returns no tool for a virtual machine (Windows 365 Cloud PC / Azure / Hyper-V / VMware / VirtualBox — a Cloud PC reports "Microsoft" and used to mis-match Surface) and only offers Surface when the model actually says "Surface"; VMs show "no OEM firmware/driver updates (use Windows Update)".
- **Microsoft Store** — "Open" + the **pending Store-updates list** (`StoreUpdatesInfo`, 0.17.57; queue merge 0.19.11): AppInstallManager with `AppUpdateOptions.AutomaticallyDownloadAndInstallUpdateIfFound = false` — a read-only search that lists UWP/MSIX updates *without* starting downloads or disturbing in-progress ones. **A fresh search misses updates the Store already queued itself** (stuck `ReadyToDownload`, or `0x80073D02` "app was in use") — those only appear in `AppInstallItems`, so the scan merges both and shows each queued item's state ("name — waiting — app was in use, retries when it closes"). Update All's Store task likewise picks queued items up, `Restart()`s any in Error/Paused/ReadyToDownload, and decodes the app-in-use HRESULT in its log.

All sources **auto-scan in parallel on open** (`EnsureLoadedAsync`, guarded, also **pre-cached from startup** — see [[Overview]]). Then:

- **Update All** (**not** tech-gated as of 0.17.64 — updating is encouraged, not destructive; nothing on this page is gated): Defender signatures → Windows Update (PSWindowsUpdate) → manufacturer tool → `winget upgrade` → Microsoft Store (WinRT) → Office C2R (launches Office's own visible updater). Per-task status rows **and the live output log live inside this card** (0.17.56 — the old always-visible bottom log block is gone; the log appears only while Update All runs). **Missing vendor tools are auto-installed** (0.17.63, matching the HUS script): Dell → winget-installs Dell Command Update then runs dcu-cli; HP → downloads HPIA from HP into hardened `C:\PCI\Tools`; Lenovo → LSUClient self-installs. GUI fallbacks only when the install/download fails; auto-installs are activity-logged. Only the passive scan-on-open never installs.
- **Windows Update history** (Microsoft.Update.Session COM, QFE fallback) with a "More…" popup.
- Warranty auto-pull remains deferred (NinjaOne does it server-side; standalone needs a Dell TechDirect API key).
