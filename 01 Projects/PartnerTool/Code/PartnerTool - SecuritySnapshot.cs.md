---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SecuritySnapshot.cs
---

# PartnerTool\SecuritySnapshot.cs

```csharp
using Microsoft.Win32;
using System.Management;

namespace PartnerTool;

public record SecuritySnapshot(
    string AvName,
    bool   AvRtpEnabled,
    bool   FwDomain,
    bool   FwPrivate,
    bool   FwPublic,
    string BitLocker,
    bool?  Activated,
    bool?  SecureBoot,
    bool?  UacEnabled
)
{
    public static SecuritySnapshot Collect()
    {
        string avName = "None detected"; bool avRtp = false;
        try
        {
            using var q = new ManagementObjectSearcher(
                @"root\SecurityCenter2",
                "SELECT displayName,productState FROM AntiVirusProduct");
            foreach (ManagementObject o in q.Get())
            {
                avName = o["displayName"]?.ToString() ?? "Unknown";
                avRtp  = ((Convert.ToInt32(o["productState"]) >> 12) & 0xF) != 0;
                break;
            }
        }
        catch { }

        static bool Fw(string sub)
        {
            try
            {
                using var k = Registry.LocalMachine.OpenSubKey(
                    $@"SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\{sub}");
                return k?.GetValue("EnableFirewall") is int v && v == 1;
            }
            catch { return false; }
        }

        string bl = "Unknown";
        try
        {
            using var q = new ManagementObjectSearcher(
                @"root\cimv2\security\MicrosoftVolumeEncryption",
                "SELECT ProtectionStatus FROM Win32_EncryptableVolume WHERE DriveLetter='C:'");
            foreach (ManagementObject o in q.Get())
            {
                bl = Convert.ToInt32(o["ProtectionStatus"]) == 1 ? "Encrypted" : "Not encrypted";
                break;
            }
        }
        catch (UnauthorizedAccessException) { bl = "Requires admin"; }
        catch { }

        bool? activated = null;
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT LicenseStatus FROM SoftwareLicensingProduct WHERE PartialProductKey IS NOT NULL AND ApplicationId='55c92734-d682-4d71-983e-d6ec3f16059f'");
            foreach (ManagementObject o in q.Get())
            {
                activated = Convert.ToInt32(o["LicenseStatus"]) == 1;
                break;
            }
        }
        catch { }

        bool? sb = null;
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(
                @"SYSTEM\CurrentControlSet\Control\SecureBoot\State");
            if (k?.GetValue("UEFISecureBootEnabled") is int v) sb = v == 1;
        }
        catch { }

        bool? uac = null;
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System");
            if (k?.GetValue("EnableLUA") is int v) uac = v == 1;
        }
        catch { }

        return new SecuritySnapshot(avName, avRtp,
            Fw("DomainProfile"), Fw("StandardProfile"), Fw("PublicProfile"),
            bl, activated, sb, uac);
    }
}
```
