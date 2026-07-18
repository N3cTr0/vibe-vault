---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SecurityAudit.cs
---

# PartnerTool\SecurityAudit.cs

```csharp
using System.Management;
using Microsoft.Win32;

namespace PartnerTool;

public enum AuditLevel { Good, Warn, Bad, Info }

/// <summary>
/// Where to send the tech to change a setting. <paramref name="Target"/> is either a
/// System32 tool/applet (launched by absolute path — we run elevated) or, when
/// <paramref name="ShellExecute"/> is true, an ms-settings: URI or an .msc console.
/// </summary>
public record AuditFix(string Tooltip, string Target, string? Args = null, bool ShellExecute = false);

public record AuditItem(string Name, AuditLevel Level, string Detail, AuditFix? Fix = null);

/// <summary>
/// A one-screen hardening scorecard: common misconfigurations a tech should check on a
/// client machine (RDP exposure, SMBv1, UAC, autologon, stale built-in accounts, Secure
/// Boot, BitLocker, firewall, PowerShell policy). All read-only registry / WMI checks.
/// </summary>
public static class SecurityAudit
{
    public static List<AuditItem> Collect()
    {
        var items = new List<AuditItem>();

        // UAC
        int? lua = RegInt(Registry.LocalMachine, @"SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System", "EnableLUA");
        items.Add(new("User Account Control (UAC)",
            lua == 1 ? AuditLevel.Good : AuditLevel.Bad,
            lua == 1 ? "Enabled" : "Disabled — strongly recommend enabling",
            new AuditFix("Open User Account Control settings", "UserAccountControlSettings.exe")));

        // RDP
        int? denyRdp = RegInt(Registry.LocalMachine, @"SYSTEM\CurrentControlSet\Control\Terminal Server", "fDenyTSConnections");
        bool rdpOn = denyRdp == 0;
        items.Add(new("Remote Desktop (RDP)",
            rdpOn ? AuditLevel.Warn : AuditLevel.Good,
            rdpOn ? "Enabled — ensure it's intended and restricted" : "Disabled",
            new AuditFix("Open Remote Desktop settings", "ms-settings:remotedesktop", ShellExecute: true)));
        if (rdpOn)
        {
            int? nla = RegInt(Registry.LocalMachine,
                @"SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp", "UserAuthentication");
            items.Add(new("RDP Network Level Authentication",
                nla == 1 ? AuditLevel.Good : AuditLevel.Bad,
                nla == 1 ? "Required (good)" : "Not required — enable NLA",
                new AuditFix("Open Remote settings (System Properties)", "SystemPropertiesRemote.exe")));
        }

        // SMBv1 — modern Windows uninstalls the SMB 1.0 feature by default, which removes the
        // mrxsmb10 client driver and leaves NO LanmanServer\...\SMB1 value. A missing value is
        // not "enabled" (that was the old check's false positive). Flag it only when the server
        // has SMB1 explicitly on, or the SMB1 client driver is installed and not disabled
        // (Start != 4). Both absent ⇒ feature not present ⇒ good.
        int? smb1Srv  = RegInt(Registry.LocalMachine, @"SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters", "SMB1");
        int? mrxStart = RegInt(Registry.LocalMachine, @"SYSTEM\CurrentControlSet\Services\mrxsmb10", "Start");
        bool smb1On = smb1Srv == 1 || (mrxStart is int ms && ms != 4);
        items.Add(new("SMBv1 protocol",
            smb1On ? AuditLevel.Bad : AuditLevel.Good,
            smb1On ? "Enabled — disable SMBv1 (legacy, insecure)" : "Not installed / disabled (good)",
            new AuditFix("Open Windows Features to turn SMB 1.0 off", "OptionalFeatures.exe")));

        // PowerShell execution policy (machine)
        var ps = RegStr(Registry.LocalMachine,
            @"SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell", "ExecutionPolicy");
        items.Add(new("PowerShell execution policy",
            ps is "Bypass" or "Unrestricted" ? AuditLevel.Warn : AuditLevel.Info,
            string.IsNullOrEmpty(ps) ? "Not set (Windows default)" : ps));

        // Autologon with stored password
        var auto = RegStr(Registry.LocalMachine, @"SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon", "AutoAdminLogon");
        var defPwd = RegStr(Registry.LocalMachine, @"SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon", "DefaultPassword");
        bool autoLogon = auto == "1" && !string.IsNullOrEmpty(defPwd);
        items.Add(new("Automatic logon",
            autoLogon ? AuditLevel.Bad : AuditLevel.Good,
            autoLogon ? "Enabled with a password stored in clear text" : "Not configured",
            new AuditFix("Open User Accounts (netplwiz)", "netplwiz.exe")));

        // Built-in Administrator / Guest accounts
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, Disabled, SID FROM Win32_UserAccount WHERE LocalAccount=True");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var sid = o["SID"]?.ToString() ?? "";
                bool disabled = o["Disabled"] is bool b && b;
                if (sid.EndsWith("-500"))
                    items.Add(new("Built-in Administrator account", disabled ? AuditLevel.Good : AuditLevel.Warn,
                        disabled ? "Disabled (good)" : $"Enabled ({o["Name"]})",
                        new AuditFix("Open Local Users and Groups", "lusrmgr.msc", ShellExecute: true)));
                else if (sid.EndsWith("-501"))
                    items.Add(new("Guest account", disabled ? AuditLevel.Good : AuditLevel.Bad,
                        disabled ? "Disabled (good)" : "Enabled — disable the Guest account",
                        new AuditFix("Open Local Users and Groups", "lusrmgr.msc", ShellExecute: true)));
            }
        }
        catch { }

        // Secure Boot
        int? sb = RegInt(Registry.LocalMachine, @"SYSTEM\CurrentControlSet\Control\SecureBoot\State", "UEFISecureBootEnabled");
        items.Add(new("Secure Boot",
            sb == 1 ? AuditLevel.Good : AuditLevel.Warn,
            sb == 1 ? "Enabled" : "Disabled or unavailable"));

        // BitLocker (C:)
        items.Add(BitLockerItem());

        // Firewall profiles
        int off = new[] { "DomainProfile", "StandardProfile", "PublicProfile" }
            .Count(p => RegInt(Registry.LocalMachine,
                $@"SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\{p}", "EnableFirewall") != 1);
        items.Add(new("Windows Firewall",
            off == 0 ? AuditLevel.Good : AuditLevel.Bad,
            off == 0 ? "All profiles enabled" : $"{off} profile(s) disabled",
            new AuditFix("Open Windows Firewall settings", "control.exe", "firewall.cpl")));

        return items;
    }

    private static AuditItem BitLockerItem()
    {
        try
        {
            using var q = new ManagementObjectSearcher(
                @"root\cimv2\security\MicrosoftVolumeEncryption",
                "SELECT ProtectionStatus FROM Win32_EncryptableVolume WHERE DriveLetter='C:'");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                bool enc = Convert.ToInt32(o["ProtectionStatus"]) == 1;
                return new("BitLocker (C:)", enc ? AuditLevel.Good : AuditLevel.Warn,
                    enc ? "Encrypted" : "Not encrypted",
                    new AuditFix("Open BitLocker settings", "control.exe", "/name Microsoft.BitLockerDriveEncryption"));
            }
        }
        catch { }
        return new("BitLocker (C:)", AuditLevel.Info, "Status unavailable");
    }

    private static int? RegInt(RegistryKey root, string path, string name)
    {
        try { using var k = root.OpenSubKey(path); return k?.GetValue(name) is int v ? v : null; }
        catch { return null; }
    }

    private static string? RegStr(RegistryKey root, string path, string name)
    {
        try { using var k = root.OpenSubKey(path); return k?.GetValue(name) as string; }
        catch { return null; }
    }
}
```
