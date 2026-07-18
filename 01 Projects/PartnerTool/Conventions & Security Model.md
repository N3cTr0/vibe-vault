---
project: PartnerTool
tags: [partnertool, conventions]
---

# Conventions & Security Model

The standing rules — every change must follow these.

## Process conventions

1. **Versioning:** every change set bumps `versions.md` (newest-first) **and** `<Version>` in `PartnerTool.csproj` **and** `Product.wxs`. `0.MINOR.PATCH` pre-release scheme; 1.0.0 = production-ready. **Use the real current date on each entry** — 26 entries once carried a copy-pasted "2026-06-08" and had to be reconstructed from Event Viewer timestamps (fixed 0.17.61).
2. **American English** in all user-facing copy (analyze/optimize/canceled/color…).
3. **Never shell out to `wmic.exe`** (being removed from Win11). Use `System.Management` or `Get-CimInstance`.
4. **Every action is logged:** commands auto-log via `ProcessRunner.RunAsync`; non-CLI system changes call `ActivityLog.Action/Result` explicitly. Sensitive reveals (BitLocker key, Wi-Fi password) log too. See [[Tech Gate & Activity Log]].
5. **New collectors also go in the report:** `FullReport.GatherAsync` + a section in `ReportBuilder` — Collect Diagnostics must stay comprehensive.
6. **Docs live in this vault** going forward — decisions, gotchas, and knowledge from work sessions get captured under the [[PartnerTool]] hub.

## Elevated-app security model (LPE prevention)

The tool runs as admin, so anything it executes or trusts must not be writable by a standard user:

- **No HKCU-sourced execution** — e.g. per-user (HKCU) uninstall strings are refused; a user could plant a command that the elevated tool would run.
- **Full System32 paths** for system tools — CreateProcess searches the app folder first and ShellExecute searches the working directory, so bare names could be hijacked by a planted exe. Use `ProcessRunner.ResolveSystemExe` (CLI tools) or `ProcessRunner.ResolveSystemTool` (any extension — exe/msc/cpl — for ShellExecute launches; falls back to the Windows dir for regedit.exe).
- **`C:\PCI` is ACL-hardened** at startup (admins/SYSTEM full, users read, no inheritance) because logs and downloaded vendor tools live there.
- **Junction/reparse-point safety** in anything that deletes recursively (temp cleaner, disk usage).
- **Validate WMI/WQL inputs — don't escape them (0.19.3).** WMI object paths (`Win32_Service.Name='…'`) and WQL queries are built by string interpolation, and we run elevated, so a name/SID that breaks out of the quotes is an injection. Escaping is the wrong defence (we had *two* different, inconsistent styles — `\'` and `''`): **refuse malformed input instead**. A service name with a quote/backslash/backtick/control char is rejected (`ServicesInfo.ValidServiceName`), start mode is checked against its three legal values, and a SID must match `^S-1-\d+(-\d+)*$` before it can drive the irreversible profile delete (`SystemManagement`). Any new WMI call that interpolates a value must validate-and-reject the same way.

## Destructive-action protection

- **Tech gate:** first destructive click per session prompts for the code = `day + year` (e.g. 26 Jun 2026 → `262026`). Prompt is deliberately vague ("Enter tech code to continue."), input masked.
- **Gated actions:** full repair, CHKDSK (scan + schedule), WU reset, advanced cleanup, temp cleans, empty recycle bin, print-queue clear, icon cache, installer cleanup, Office language removal, WinHTTP reset, Adobe policy write, network resets, release/renew IP, restart/shutdown/sign-out, service stop/restart, kill process, hosts save, profile delete, app uninstall.
- **Deliberately NOT gated:** the entire Updates section incl. Update All (winget/Dell/Store/Office) — installing updates is encouraged, not destructive; users can do the same via Windows themselves (0.17.64). Still activity-logged.
- **Servicing lock (0.17.68):** CBS/TrustedInstaller-heavy operations (Full Repair, Advanced Cleanup, CHKDSK, WU Reset, Update All) share the app-wide `ServicingLock` — two at once make one fail (observed: DISM RestoreHealth 0x800F0915 during Update All). Any new fix that drives DISM/SFC/Windows Update must acquire it (`BeginServicing` on RepairPage).
- **Hard blocks (no override in-tool):**
  - Services: core Windows (RpcSs, DcomLaunch, LSM, EventLog, SamSs…), firewall (mpssvc, BFE), Defender + EDR agents (SentinelOne, Huntress, CrowdStrike, Cylance) — enforced at UI *and* in `ServicesInfo.Control`.
  - Processes: BSOD-critical set + **svchost** (can't tell which services an instance hosts).
  - Startup items: security software (Defender/SecurityHealth, SentinelOne, Huntress, CrowdStrike, Sophos, Malwarebytes, ESET, Kaspersky, McAfee/Trellix, Cortex) can't be disabled — UI check + `StartupInfo.SetEnabled` backstop.
