---
project: PartnerTool
tags: [partnertool, feature]
---

# Security (page)

- **Hardening Scorecard** (`SecurityAudit.cs`): UAC, RDP (+NLA when on), SMBv1 (by `mrxsmb10` driver / explicit server value — absent ≠ enabled!), PowerShell execution policy, autologon-with-stored-password, built-in Administrator/Guest, Secure Boot, BitLocker, firewall profiles. **Values are clickable** for anything changeable from Windows — each opens the right applet (UAC settings, ms-settings:remotedesktop, SystemPropertiesRemote, OptionalFeatures, netplwiz, lusrmgr.msc, BitLocker CP, firewall.cpl) via elevated-safe absolute paths. PowerShell policy + Secure Boot stay plain text (no Windows UI to change them).
- **BitLocker recovery key** viewer (activity-logged reveal).
- **Microsoft Defender** card: RTP, tamper protection, signature version/age, scan ages, threat history — hides when third-party AV is active.
