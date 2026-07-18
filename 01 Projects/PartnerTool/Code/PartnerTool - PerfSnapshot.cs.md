---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PerfSnapshot.cs
---

# PartnerTool\PerfSnapshot.cs

```csharp
using System.IO;
using System.Management;

namespace PartnerTool;

public record PerfSnapshot(
    string   CpuName,
    int      Cores,
    int      Threads,
    double   RamUsedGb,
    double   RamTotalGb,
    double   RamPct,
    double   DiskUsedGb,
    double   DiskTotalGb,
    double   DiskPct,
    TimeSpan Uptime,
    DateTime LastBoot,
    int?     BatteryPct,
    bool?    OnAcPower,
    int      CurrentClockMhz,
    int      MaxClockMhz
)
{
    /// <summary>Rated/base clock as text, e.g. "2.80 GHz". Empty if unknown.</summary>
    public string MaxClockText => MaxClockMhz > 0 ? $"{MaxClockMhz / 1000.0:F2} GHz" : "";

    /// <summary>
    /// Current vs rated clock — a rough throttle hint. WMI's CurrentClockSpeed often tracks the
    /// power/thermal cap, so a current well below the rated max can indicate throttling.
    /// </summary>
    public string ClockText => MaxClockMhz <= 0
        ? "Unknown"
        : $"{CurrentClockMhz / 1000.0:F2} GHz of {MaxClockMhz / 1000.0:F2} GHz";

    public bool LikelyThrottled => MaxClockMhz > 0 && CurrentClockMhz > 0 && CurrentClockMhz < MaxClockMhz * 0.7;

    /// <summary>False when the boot time couldn't be read (WMI unavailable).</summary>
    public bool   BootKnown  => LastBoot != DateTime.MinValue;

    /// <summary>Formatted uptime, or "Unknown" when the boot time is unavailable.</summary>
    public string UptimeText => BootKnown
        ? $"{(int)Uptime.TotalDays}d {Uptime.Hours}h {Uptime.Minutes}m"
        : "Unknown";

    public static PerfSnapshot Collect()
    {
        string cpuName = "Unknown"; int cores = 0, threads = 0, curClock = 0, maxClock = 0;
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name,NumberOfCores,NumberOfLogicalProcessors,CurrentClockSpeed,MaxClockSpeed FROM Win32_Processor");
            foreach (ManagementObject o in q.Get())
            {
                cpuName = o["Name"]?.ToString()?.Trim() ?? "Unknown";
                cores   = Convert.ToInt32(o["NumberOfCores"]);
                threads = Convert.ToInt32(o["NumberOfLogicalProcessors"]);
                if (o["CurrentClockSpeed"] is { } cc) curClock = Convert.ToInt32(cc);
                if (o["MaxClockSpeed"]     is { } mc) maxClock = Convert.ToInt32(mc);
                break;
            }
        }
        catch { }

        double ramTotal = 0, ramUsed = 0, ramPct = 0;
        TimeSpan uptime  = TimeSpan.Zero;
        DateTime lastBoot = DateTime.MinValue;
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT TotalVisibleMemorySize,FreePhysicalMemory,LastBootUpTime FROM Win32_OperatingSystem");
            foreach (ManagementObject o in q.Get())
            {
                ramTotal  = Convert.ToDouble(o["TotalVisibleMemorySize"]) / 1048576.0;
                double free = Convert.ToDouble(o["FreePhysicalMemory"]) / 1048576.0;
                ramUsed   = ramTotal - free;
                ramPct    = ramTotal > 0 ? ramUsed / ramTotal * 100 : 0;
                lastBoot  = ManagementDateTimeConverter.ToDateTime(o["LastBootUpTime"]?.ToString() ?? "");
                uptime    = DateTime.Now - lastBoot;
                break;
            }
        }
        catch { }

        double diskTotal = 0, diskUsed = 0, diskPct = 0;
        try
        {
            var d = new DriveInfo("C");
            diskTotal = d.TotalSize / 1073741824.0;
            double free = d.AvailableFreeSpace / 1073741824.0;
            diskUsed  = diskTotal - free;
            diskPct   = diskTotal > 0 ? diskUsed / diskTotal * 100 : 0;
        }
        catch { }

        int? battPct = null; bool? onAc = null;
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT EstimatedChargeRemaining,BatteryStatus FROM Win32_Battery");
            foreach (ManagementObject o in q.Get())
            {
                if (o["EstimatedChargeRemaining"] is { } pct)
                    battPct = Convert.ToInt32(pct);
                // Win32_Battery.BatteryStatus: 1 = discharging (running on battery).
                // Every other value (2 = AC, 3 = fully charged, 6-9 = charging) means
                // mains power is connected.
                if (o["BatteryStatus"] is { } status)
                    onAc = Convert.ToInt32(status) != 1;
                break;
            }
        }
        catch { }

        return new PerfSnapshot(cpuName, cores, threads,
            ramUsed, ramTotal, ramPct,
            diskUsed, diskTotal, diskPct,
            uptime, lastBoot, battPct, onAc,
            curClock, maxClock);
    }
}
```
