---
project: PartnerTool
tags: [partnertool, deep-dive, repair]
---

# Repair Internals

The non-obvious mechanics behind the [[Repair]] page's specialty fixes.

## Windows Installer cleanup (Adobe bloat)

`InstallerCleanup.cs`. Adobe Acrobat/Reader's updater caches monthly patches in `C:\Windows\Installer` and never purges superseded ones (30–300+ GB observed).

- **Safety:** ask Windows Installer itself (`WindowsInstaller.Installer` COM — `Products`/`Patches` → `LocalPackage`) for the authoritative keep-set; delete **only** top-level `.msi/.msp` not in that set. Abort if the keep-set is empty/unreadable. Same method as PatchCleaner. Never touch `{GUID}` subfolders.
- **Descriptions:** each orphan's product is read read-only (MSI `Property` table: ProductName/Manufacturer; MSP `SummaryInformation` Subject) so logs say *what* each hex-named file was.
- **Prevention:** `PatchCleanFlag=1` (DWORD) under both Adobe `FeatureLockdown` policy keys — Adobe's own fix; updater then purges old patches.
- Workflow buttons: 1. Prevent Recurrence → 2. Scan (dry-run preview, by-product breakdown) → 3. Clean.

## Office language pack removal

`OfficeLanguages.cs`. Click-to-Run installs register **one ARP entry per language**; uninstall string contains `culture=xx-xx` (display names are localized — *never* match on the name). Removal = run that entry's own `OfficeClickToRun.exe` command + `DisplayLevel=False forceappshutdown=True`, then verify against a re-scan (not the exit code). Keeps **all** `en-*`; refuses to run if no English pack exists (Office must keep ≥1 language).

## Temp cleaner (all users)

`TempCleaner.cs`. Per profile: `AppData\Local\Temp`, `CrashDumps`, `WER`, `INetCache`, `D3DSCache`, RDP cache; plus `C:\Windows\Temp`. Explicitly **not** browser caches or the Office document cache (user-approved scope). Skips reparse points (junction-safe for an elevated delete), skips in-use files, keeps folders, logs per-folder breakdown + skipped files. Production incident note: clearing CrashDumps/WER destroys crash-diagnostic evidence — the only irreversible loss.

## WPAD / proxy fix (SonicWall breaks Outlook)

`ProxyRepair.cs`. "Automatically detect settings" is **bit 0x08 of the flags byte at offset 8** in the binary `DefaultConnectionSettings`/`SavedLegacySettings` blobs (HKCU Connections key) — no friendly registry value exists. Toggle off→on, bump the change counter (LE uint at offset 4), then `InternetSetOption(SETTINGS_CHANGED=39, REFRESH=37)`. Companion: `netsh winhttp reset proxy` (gated — can break proxied environments).

## Full repair sequence

DISM CheckHealth → ScanHealth → RestoreHealth **then** SFC /scannow — deliberate order: SFC repairs from the component store DISM just restored. DISM progress bars are parsed to % by `ProcessRunner`; every step gets a heartbeat ticker so long silences don't look frozen.
