---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DellRemediation.cs
---

# PartnerTool\DellRemediation.cs

```csharp
using System.IO;
using System.Management;

namespace PartnerTool;

/// <summary>
/// What Dell SupportAssist OS Recovery's "System Repair" is costing this machine, plus the state of
/// VSS shadow storage on the system drive.
///
/// TWO SEPARATE THINGS, routinely confused:
///
///  1. C:\ProgramData\Dell\SARemediation\SystemRepair\Snapshots — Dell's own repair points. NOT VSS.
///     The default reserve is 15 GB, Dell staff call 20–25 GB "normal", and a known purge bug lets it
///     reach 85–100 GB+. Dell's guidance is explicit: do NOT delete the folder by hand (it breaks the
///     repair feature) — turn System Repair OFF in SupportAssist and it purges itself on reboot.
///     So we only REPORT this, and point the tech at the SupportAssist setting.
///
///  2. VSS shadow storage — Dell KB 000129138: VSS ships with MaxSize unset, so when SupportAssist OS
///     Recovery reads VSS the service goes UNBOUNDED and quietly fills the drive with shadow copies
///     (the classic "unaccounted-for space"). Dell's own fix is to bound it:
///         vssadmin resize shadowstorage /For=C: /On=C: /MaxSize=2%
///     That we CAN do safely, so it's offered as a fix.
///
/// Reading the folder size needs elevation (ProgramData\Dell is ACL'd) — the app runs elevated.
/// </summary>
public static class DellRemediation
{
    public const string RootPath      = @"C:\ProgramData\Dell\SARemediation";
    public const string SnapshotPath  = @"C:\ProgramData\Dell\SARemediation\SystemRepair\Snapshots";

    /// <summary>
    /// The cap we set when bounding runaway shadow storage. Dell's KB suggests 2%; we use 5% as our
    /// house default — it stops the runaway growth while leaving more room for real restore points.
    /// </summary>
    public const int VssMaxPercent = 5;

    // "Normal" per Dell is 20–25 GB, so warn above that and call it bad once it's clearly the bug.
    private const double WarnGb = 25.0;
    private const double BadGb  = 40.0;

    public class DellInfo
    {
        public bool IsDell        { get; init; }
        public bool FolderExists  { get; init; }
        public long TotalBytes    { get; init; }   // whole SARemediation tree
        public long SnapshotBytes { get; init; }   // just SystemRepair\Snapshots

        public bool VssQueried       { get; init; }
        public bool VssHasAssociation{ get; init; }   // false → no shadow storage set up (System Protection off)
        public long VssMaxBytes      { get; init; }   // -1 → UNBOUNDED
        public long VssUsedBytes     { get; init; }
        public long VolumeBytes      { get; init; }

        public double TotalGb    => TotalBytes    / 1073741824.0;
        public double SnapshotGb => SnapshotBytes / 1073741824.0;
        public double VssUsedGb  => VssUsedBytes  / 1073741824.0;

        /// <summary>
        /// A shadow-storage association EXISTS and is uncapped (or capped so high it may as well be) —
        /// the actual Dell KB 000129138 problem. A machine with no association at all (System
        /// Protection off) is NOT flagged: that's a normal, healthy state, not runaway shadow storage.
        /// </summary>
        public bool VssUnbounded => VssQueried && VssHasAssociation &&
                                    (VssMaxBytes < 0 || (VolumeBytes > 0 && VssMaxBytes >= VolumeBytes / 2));

        /// <summary>Shadow storage is set up with a sane limit — nothing to do.</summary>
        public bool VssConfigured => VssQueried && VssHasAssociation && !VssUnbounded;

        public bool Bloated  => TotalGb >= WarnGb;
        public bool VeryBad  => TotalGb >= BadGb;

        public string VssMaxText => !VssQueried        ? "unknown"
                                  : !VssHasAssociation ? "not configured (System Protection off)"
                                  : VssMaxBytes < 0     ? "UNBOUNDED"
                                  : $"{VssMaxBytes / 1073741824.0:F1} GB";
    }

    /// <summary>
    /// Cheap check (manufacturer string + folder existence, no directory walk) for whether the Dell
    /// card should be shown at all. Lets the UI decide visibility without paying for the full
    /// <see cref="Collect"/> sizing until the tech actually asks for it.
    /// </summary>
    public static bool IsApplicable() =>
        SystemInfo.GetManufacturer().Contains("Dell", StringComparison.OrdinalIgnoreCase) &&
        Directory.Exists(RootPath);

    public static DellInfo Collect()
    {
        bool isDell = SystemInfo.GetManufacturer().Contains("Dell", StringComparison.OrdinalIgnoreCase);
        bool exists = Directory.Exists(RootPath);

        long total = 0, snaps = 0;
        if (exists)
        {
            total = DirSize(RootPath);
            snaps = Directory.Exists(SnapshotPath) ? DirSize(SnapshotPath) : 0;
        }

        var (queried, hasAssoc, max, used, volume) = ShadowStorage("C:");

        return new DellInfo
        {
            IsDell = isDell, FolderExists = exists,
            TotalBytes = total, SnapshotBytes = snaps,
            VssQueried = queried, VssHasAssociation = hasAssoc,
            VssMaxBytes = max, VssUsedBytes = used, VolumeBytes = volume,
        };
    }

    /// <summary>
    /// Shadow-storage limits for a drive, straight from WMI (no wmic). <paramref name="queried"/> is
    /// false only if the query itself couldn't run. <paramref name="hasAssoc"/> is false when the
    /// volume has NO shadow-storage association — System Protection is simply off, a normal state we
    /// deliberately don't treat as the Dell runaway. MaxBytes = -1 means VSS reports UNBOUNDED.
    /// </summary>
    private static (bool queried, bool hasAssoc, long max, long used, long volume) ShadowStorage(string driveLetter)
    {
        try
        {
            // Win32_ShadowStorage.Volume is a path reference containing the volume's DeviceID, so
            // resolve C:'s DeviceID first and match on it.
            string deviceId = "", capacity = "";
            using (var vols = new ManagementObjectSearcher(
                       $"SELECT DeviceID, Capacity FROM Win32_Volume WHERE DriveLetter = '{driveLetter}'"))
                foreach (ManagementObject v in vols.Get())
                {
                    deviceId = v["DeviceID"]?.ToString() ?? "";
                    capacity = v["Capacity"]?.ToString() ?? "";
                    break;
                }
            if (deviceId.Length == 0) return (false, false, 0, 0, 0);

            long volumeBytes = long.TryParse(capacity, out var cap) ? cap : 0;

            using var ss = new ManagementObjectSearcher("SELECT MaxSpace, UsedSpace, Volume FROM Win32_ShadowStorage");
            foreach (ManagementObject o in ss.Get())
            {
                var volRef = o["Volume"]?.ToString() ?? "";
                // The DeviceID is embedded (escaped) in the reference string; compare on the GUID.
                if (!ReferencesVolume(volRef, deviceId)) continue;

                ulong max  = ToULong(o["MaxSpace"]);
                ulong used = ToULong(o["UsedSpace"]);
                // VSS reports "no limit" as the max value of the field.
                long maxBytes = max >= ulong.MaxValue || max > long.MaxValue ? -1 : (long)max;
                return (true, true, maxBytes, (long)Math.Min(used, long.MaxValue), volumeBytes);
            }

            // No association row → no shadow storage on this volume (System Protection off). Healthy;
            // not the runaway condition, so hasAssoc = false and callers leave it alone.
            return (true, false, 0, 0, volumeBytes);
        }
        catch { return (false, false, 0, 0, 0); }
    }

    private static bool ReferencesVolume(string volRef, string deviceId)
    {
        // deviceId looks like \\?\Volume{guid}\ ; the reference escapes its backslashes.
        int a = deviceId.IndexOf('{'), b = deviceId.IndexOf('}');
        if (a < 0 || b <= a) return false;
        return volRef.Contains(deviceId[a..(b + 1)], StringComparison.OrdinalIgnoreCase);
    }

    private static ulong ToULong(object? o)
    {
        try { return o is null ? 0 : Convert.ToUInt64(o); } catch { return 0; }
    }

    /// <summary>
    /// Bound VSS shadow storage on C: to <see cref="VssMaxPercent"/>% — Dell KB 000129138's own fix.
    /// Shrinking the limit below what's currently used discards the oldest shadow copies, so this
    /// can delete existing System Restore points. Tech-gated by the caller.
    /// </summary>
    public static async Task<string> BoundVssAsync(Action<string>? log)
    {
        log?.Invoke($"Setting VSS shadow storage on C: to a {VssMaxPercent}% cap…");
        ActivityLog.Action("Dell", $"Bound VSS shadow storage on C: to {VssMaxPercent}%");

        var output = await ProcessRunner.RunCaptureAsync(
            "vssadmin.exe", $"resize shadowstorage /For=C: /On=C: /MaxSize={VssMaxPercent}%", 60000);

        foreach (var line in output.Split('\n').Select(l => l.Trim()).Where(l => l.Length > 0))
            log?.Invoke(line);

        bool ok = output.Contains("Successfully resized", StringComparison.OrdinalIgnoreCase);
        var result = ok
            ? $"VSS shadow storage on C: is now capped at {VssMaxPercent}% of the drive."
            : "vssadmin did not report success — see the log above.";
        ActivityLog.Result("Dell", result);
        return result;
    }

    /// <summary>Open the SupportAssist OS Recovery settings, where System Repair is turned off.</summary>
    public static void OpenSupportAssistSettings()
    {
        ActivityLog.Action("Dell", "Open SupportAssist OS Recovery settings");
        foreach (var target in new[]
                 {
                     @"C:\Program Files\Dell\SupportAssistAgent\bin\SupportAssistUI.exe",
                     @"C:\Program Files (x86)\Dell\SupportAssistAgent\bin\SupportAssistUI.exe",
                 })
        {
            if (!File.Exists(target)) continue;
            try
            {
                System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo(target) { UseShellExecute = true });
                return;
            }
            catch { }
        }
        // Fall back to the folder so the tech can at least see what's there.
        try
        {
            System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo(
                ProcessRunner.ResolveSystemExe("explorer.exe"), RootPath) { UseShellExecute = false });
        }
        catch { }
    }

    private const int MaxDepth = 32;

    private static long DirSize(string path, int depth = 0)
    {
        if (depth > MaxDepth) return 0;
        long total = 0;
        try
        {
            foreach (var f in Directory.EnumerateFiles(path))
                try { total += new FileInfo(f).Length; } catch { }

            foreach (var d in Directory.EnumerateDirectories(path))
            {
                try
                {
                    if ((File.GetAttributes(d) & FileAttributes.ReparsePoint) != 0) continue;
                    total += DirSize(d, depth + 1);
                }
                catch { }
            }
        }
        catch { }
        return total;
    }
}
```
