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
    /// has a RecoveryPassword protector - the same data <c>manage-bde -protectors -get</c>
    /// returns. Requires admin (the app runs elevated). Returns an empty list when none
    /// exist (drive unencrypted, or TPM-only with no recovery password) or on error.
    /// </summary>
    /// <summary>
    /// Cheap "is this card worth showing" check. <see cref="GetRecoveryKeys"/> costs a
    /// GetKeyProtectors + GetKeyProtectorNumericalPassword round trip per volume per protector,
    /// which is slow on an encrypted machine - too slow to run just to decide a Visibility.
    /// The window reads the actual keys on demand and says so when there are none.
    /// </summary>
    public static bool AnyProtectedVolume()
    {
        try
        {
            using var q = new ManagementObjectSearcher(
                @"root\cimv2\Security\MicrosoftVolumeEncryption",
                "SELECT ProtectionStatus FROM Win32_EncryptableVolume");
            foreach (ManagementObject v in q.Get())
                using (v)
                    if (Convert.ToInt32(v["ProtectionStatus"]) == 1) return true;
        }
        catch { }
        return false;
    }

    public static List<BitLockerKey> GetRecoveryKeys()
    {
        var keys = new List<BitLockerKey>();
        try
        {
            // SELECT * on purpose. A projection comes back as a partial instance with an empty
            // __PATH, and InvokeMethod on one throws "Operation is not valid due to the current
            // state of the object" - which the per-volume catch below swallowed, so this returned
            // an empty list on every machine and the window said no key was stored. Same trap as
            // Win32_Printer.Delete().
            using var searcher = new ManagementObjectSearcher(
                @"root\cimv2\Security\MicrosoftVolumeEncryption",
                "SELECT * FROM Win32_EncryptableVolume");

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
                                    string.IsNullOrEmpty(drive) ? "-" : drive,
                                    id.Trim('{', '}'),
                                    pwd));
                        }
                        catch (Exception ex) { ActivityLog.Result("BitLocker", $"key read failed on {drive}: {ex.Message}"); }
                    }
                }
                catch (Exception ex) { ActivityLog.Result("BitLocker", $"protector read failed on {drive}: {ex.Message}"); }
            }
        }
        catch (Exception ex) { ActivityLog.Result("BitLocker", $"volume enumeration failed: {ex.Message}"); }
        return keys;
    }
}
```
