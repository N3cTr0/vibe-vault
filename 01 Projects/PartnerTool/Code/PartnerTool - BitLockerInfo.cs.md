---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BitLockerInfo.cs
---

# PartnerTool\BitLockerInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record BitLockerKey(string Drive, string Identifier, string RecoveryPassword);

public static class BitLockerInfo
{
    /// <summary>
    /// Reads the 48-digit numerical recovery passwords for every BitLocker volume that
    /// has a RecoveryPassword protector — the same data <c>manage-bde -protectors -get</c>
    /// returns. Requires admin (the app runs elevated). Returns an empty list when none
    /// exist (drive unencrypted, or TPM-only with no recovery password) or on error.
    /// </summary>
    public static List<BitLockerKey> GetRecoveryKeys()
    {
        var keys = new List<BitLockerKey>();
        try
        {
            using var searcher = new ManagementObjectSearcher(
                @"root\cimv2\Security\MicrosoftVolumeEncryption",
                "SELECT DriveLetter FROM Win32_EncryptableVolume");

            foreach (ManagementObject vol in searcher.Get())
            {
                string drive = vol["DriveLetter"]?.ToString() ?? "";
                try
                {
                    // KeyProtectorType 3 = Numerical Password (the recovery key).
                    var inParams = vol.GetMethodParameters("GetKeyProtectors");
                    inParams["KeyProtectorType"] = (uint)3;
                    var outParams = vol.InvokeMethod("GetKeyProtectors", inParams, null);
                    if (outParams?["VolumeKeyProtectorID"] is not string[] ids) continue;

                    foreach (var id in ids)
                    {
                        try
                        {
                            var inP = vol.GetMethodParameters("GetKeyProtectorNumericalPassword");
                            inP["VolumeKeyProtectorID"] = id;
                            var outP = vol.InvokeMethod("GetKeyProtectorNumericalPassword", inP, null);
                            var pwd = outP?["NumericalPassword"]?.ToString();
                            if (!string.IsNullOrWhiteSpace(pwd))
                                keys.Add(new BitLockerKey(
                                    string.IsNullOrEmpty(drive) ? "—" : drive,
                                    id.Trim('{', '}'),
                                    pwd));
                        }
                        catch { }
                    }
                }
                catch { }
            }
        }
        catch { }
        return keys;
    }
}
```
