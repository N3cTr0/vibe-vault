---
project: PartnerTool
tags: [partnertool, feature]
---

# Repair (page)

Card order: Quick Fixes → System Restore Points → Reports → **Collect Diagnostics** (pinned with Reports — it's a report, not a repair; 0.17.66) → **the heavy fixes sorted alphabetically** (Advanced Cleanup, Check Disk, Clean Temp, [Dell — Dell HW only], Feature-Update Leftovers, Office Language Packs, Proxy/WPAD, System File & Image Repair, Windows Installer Cleanup, Windows Update Reset). Every card gets its **own inline log** (hidden until that fix runs; also saved to `C:\PCI\Logs\PartnerTool_<Tag>_….log`). Internals of the specialty fixes: [[Repair Internals]].

**Disk Cleanup (`cleanmgr`) coverage** (why we don't ship/launch it — 0.19.1): the tool covers Disk Cleanup's meaningful categories across several cards, and across *all* user profiles (which cleanmgr doesn't). Temp/INetCache/D3DSCache/CrashDumps/WER → **Clean Temp Files**; Recycle Bin → **Empty Recycle Bin**; thumbnails → **Clear Icon Cache**; superseded WinSxS components ("Windows Update Cleanup") → **Advanced Cleanup**; SoftwareDistribution download cache → **Windows Update Reset**; **previous-Windows / upgrade staging + Delivery Optimization** → **Feature-Update Leftovers** (0.19.1, the last real gap). Deliberately NOT done: browser caches / Office document cache (MSP liability — same reasoning as the Health Check research in [[Lessons Learned]]); legacy "Downloaded Program Files" (always empty on modern Windows).

| Card | Notes |
|---|---|
| Quick Fixes | Empty recycle bin*, Restart/clear print spooler (*clear queue gated*), Restart Explorer/Audio, Re-register Store, Clear icon cache*, Memory diagnostic* (*gated). "Clear Temp Files" retired 0.17.66 — duplicated Clean Temp Files (All Users) |
| System Restore Points | **Create Restore Point** (moved here 0.17.62; own status + inline log, list auto-refreshes) + list + open wizard |
| Reports | Battery report, Group Policy report (HTML → browser) |
| System File & Image Repair | DISM CheckHealth→ScanHealth→RestoreHealth→SFC, live % + per-step status (gated) |
| Check Disk | online `chkdsk C: /scan` + **Schedule /f /r** at next boot (both gated) |
| Proxy / WPAD | re-enable auto-detect (SonicWall/Outlook fix) + reset WinHTTP proxy (gated) |
| Clean Temp Files (all users) | **Scan** (read-only dry-run, shows per-folder usage + total) then **Clean** (gated). See [[Repair Internals]] — approved folder scope, junction-safe |
| Windows Installer Cleanup (Adobe) | prevent-recurrence + scan preview + clean orphans (gated) |
| Office Language Packs | remove non-English C2R languages, keep all `en-*` (gated) |
| Windows Update Reset | stop services, rename SoftwareDistribution/catroot2, restart (gated) |
| Advanced Cleanup | DISM StartComponentCleanup + WMI verify/salvage (gated) |
| Feature-Update Leftovers | **0.19.1.** Reclaims Disk Cleanup's big categories the temp cleaner skips: `Windows.old`, `$Windows.~BT`/`~WS`, `$GetCurrent`, Delivery Optimization cache. **Scan** sizes what's present (read-only, reparse-safe); **Clean** gated + confirmed (Windows.old/staging removal is **permanent** — ends the ~10-day rollback). ACL'd folders removed via takeown+icacls (Administrators SID `*S-1-5-32-544`, locale-independent) then recursive delete; DO via `Delete-DeliveryOptimizationCache`. Re-measures after. Engine `FeatureUpdateCleanup`. |
| Dell SupportAssist (System Repair) | **0.19.0, Dell hardware only** (card `CardDell` stays collapsed otherwise). **Scans on demand (0.19.4):** a cheap `IsApplicable` check (maker + folder-exists, no walk) shows the card with a **Scan** button; the multi-GB SARemediation sizing + VSS check only runs on click, then the button becomes **Refresh** — opening the Repair page no longer triggers a folder walk. **Cap Shadow Storage** = Dell's KB 000129138 fix (`vssadmin … /MaxSize=5%` — 5% house default vs Dell's 2%; `DellRemediation.VssMaxPercent`), confirmed + gated, discards restore points. **Open SupportAssist** launches the OS Recovery UI (where System Repair is turned off — the only Dell-supported way to purge the snapshot folder; we never delete it ourselves). Engine `DellRemediation`; also a [[Health Check]] finding. |
| Collect Diagnostics | full everything-report .zip — see [[Updates]]/report convention in [[Conventions & Security Model]] |
