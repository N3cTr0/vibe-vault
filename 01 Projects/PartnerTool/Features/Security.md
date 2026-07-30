---
project: PartnerTool
tags: [partnertool, feature]
---

# Security (page)

- **Hardening Scorecard** (`SecurityAudit.cs`): UAC, RDP (+NLA when on), SMBv1 (by `mrxsmb10` driver / explicit server value — absent ≠ enabled!), PowerShell execution policy, autologon-with-stored-password, built-in Administrator/Guest, Secure Boot, BitLocker, firewall profiles. **Values are clickable** for anything changeable from Windows — each opens the right applet (UAC settings, ms-settings:remotedesktop, SystemPropertiesRemote, OptionalFeatures, netplwiz, lusrmgr.msc, BitLocker CP, firewall.cpl) via elevated-safe absolute paths. PowerShell policy + Secure Boot stay plain text (no Windows UI to change them).
  **PowerShell execution policy is fixable in place** (0.20.x): when it reads Bypass/Unrestricted the value becomes a link that clears the machine `ExecutionPolicy` in both registry views — `Set-ExecutionPolicy Undefined` without a shell. There's no Windows settings page for it, so it uses a sentinel `AuditFix.Target` that `SecurityPage.Fix_Click` special-cases.
- **BitLocker recovery key** viewer (activity-logged reveal). **The card only appears when a volume is actually protected** — it used to decide that by reading every recovery key on the machine, which is the single most expensive call on the page (see below).
- **Microsoft Defender** card: RTP, tamper protection, signature version/age, scan ages, threat history — hides when third-party AV is active. **Open Defender** button, and a non-zero threat count is a link (0.24.9). The link goes to `windowsdefender://threat` (Virus & threat protection, which carries the Protection history link) — `windowsdefender://protectionhistory` is **not a recognised URI** and silently lands on Home.
- **ProSentry** card (`ProsentryInfo.cs`, 0.20.x): PCI's managed stack at a glance — Atakama (detected by the adapter's DNS pointing at its loopback resolvers `127.97.116.97/.98`), Huntress EDR, Duo, AutoElevate — plus a **Device Management** row for Intune (an `Enrollments` subkey whose `ProviderID` is `MS DM Server` with a non-zero `EnrollmentType`, falling back to the Intune Management Extension service). A green dot means the agent is *active*, not merely installed. Duo is deliberately matched on the `DuoCredProv` credential-provider subkey or its uninstall entry, **never** the bare `SOFTWARE\Duo Security` parent, which lingers empty and false-positives.

## Page load (0.24.14)

Reported from the field as "takes about a minute" on a managed machine, and not reproducible on a workgroup VM with BitLocker off. Four things changed:

- **Each card paints when its own collector returns.** All four were behind a single `Task.WhenAll`, so the page stayed blank until the slowest finished.
- **The BitLocker card checks `ProtectionStatus` instead of reading the keys.** `GetRecoveryKeys()` costs a `GetKeyProtectors` + `GetKeyProtectorNumericalPassword` round trip per volume per protector; it was being called to compute a bool for a `Visibility`. The window still reads the real keys on demand.
- **ProSentry does one `Win32_Service` enumeration and at most one uninstall-hive walk.** It was asking WMI for each agent service by name (~330 ms per round trip) and re-opening every uninstall subkey for each of three name lookups.
- **Per-collector timings go to the activity log**, so the next slow machine names its own culprit in its diagnostics bundle. Generalised into the shared `LoadTimer` across every page in 0.24.15 — see [[Architecture]].
