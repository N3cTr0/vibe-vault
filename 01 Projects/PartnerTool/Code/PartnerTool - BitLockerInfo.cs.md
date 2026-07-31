---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BitLockerInfo.cs
---

# PartnerTool\BitLockerInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record BitLockerKey(string Drive, string Identifier, string RecoveryPassword, string Backup)
{
    /// <summary>True when this key exists nowhere but the disk - nobody can retrieve it remotely.</summary>
    public bool NotEscrowed => Backup == BitLockerInfo.NotBackedUp;
}

public static class BitLockerInfo
{
    /// <summary>
    /// Reads the 48-digit numerical recovery passwords for every BitLocker volume that
    /// has a RecoveryPassword protector - the same data <c>manage-bde -protectors -get</c>
    /// returns. Requires admin (the app runs elevated). Returns an empty list when none
    /// exist (drive unencrypted, or TPM-only with no recovery password) or on error.
    /// </summary>
    public const string NotBackedUp = "Not backed up anywhere";

    /// <summary>
    /// Where a recovery password has been escrowed, from GetNumericalPasswordBackupType - the same
    /// thing manage-bde prints as "Backup type". Worth showing per key: a machine often carries a
    /// pre-enrolment protector that was never escrowed alongside the one Intune backed up, and only
    /// the escrowed one can be retrieved by anyone not standing at the machine.
    ///
    /// Read through WMI rather than parsing manage-bde on purpose - ProcessRunner audit-logs command
    /// output, and manage-bde prints the passwords themselves, which would put them in the activity
    /// log and therefore in every diagnostics bundle.
    /// </summary>
    private static string BackupOf(ManagementObject vol, string id)
    {
        try
        {
            var inP = vol.GetMethodParameters("GetNumericalPasswordBackupType");
            inP["VolumeKeyProtectorID"] = id;
            var outP = vol.InvokeMethod("GetNumericalPasswordBackupType", inP, null);
            if (outP is null || Convert.ToUInt32(outP["ReturnValue"]) != 0) return "";

            uint t = Convert.ToUInt32(outP["BackupInfoType"]);
            if (t == 0) return NotBackedUp;

            var where = new List<string>();
            if ((t & 1) != 0) where.Add("Active Directory");
            if ((t & 2) != 0) where.Add("Microsoft account");
            if ((t & 4) != 0) where.Add("Entra ID");
            // Anything outside the three documented flags: say it's escrowed, but don't invent a
            // name for it - a wrong label here is worse than an unspecific one.
            if (where.Count == 0 || (t & ~7u) != 0) where.Add($"type {t}");
            return "Escrowed to " + string.Join(", ", where);
        }
        catch { return ""; }
    }

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
                                    pwd,
                                    BackupOf(vol, id)));
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
