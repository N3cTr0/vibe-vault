---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PerfDetails.cs
---

# PartnerTool\PerfDetails.cs

```csharp
using System.Diagnostics;
using System.Management;
using System.Runtime.InteropServices;

namespace PartnerTool;

/// <summary>
/// One-shot advanced details for the Performance window (the stuff the live sampler doesn't give):
/// CPU make-up + cache + virtualization, memory type/speed/slots, the system disk, the primary NIC
/// and uptime. Reuses <see cref="HardwareInfo"/>/<see cref="NetworkInfo"/> for disk/network so we
/// don't duplicate those queries. All reads are guarded.
/// </summary>
public class PerfDetails
{
    // CPU
    public string CpuName = "—";
    public double BaseGhz;
    public int    Sockets = 1, Cores, Logical;
    public string Virtualization = "—";
    public string L1 = "—", L2 = "—", L3 = "—";
    public string Uptime = "—";

    // Memory
    public double RamTotalGb;
    public string RamType = "—";
    public int    RamSpeed;
    public string Slots = "—";

    // Disk
    public string DiskModel = "—", DiskType = "—";
    public double DiskSizeGb;

    // Network
    public string NetName = "—", NetType = "—", NetIp = "—";

    public static PerfDetails Collect()
    {
        var d = new PerfDetails();

        // ── CPU ──
        try
        {
            int sockets = 0, cores = 0, logical = 0; double maxMhz = 0; string name = ""; bool? virt = null; uint l2 = 0, l3 = 0;
            using var q = new ManagementObjectSearcher(
                "SELECT Name, MaxClockSpeed, NumberOfCores, NumberOfLogicalProcessors, L2CacheSize, L3CacheSize, VirtualizationFirmwareEnabled FROM Win32_Processor");
            foreach (ManagementObject o in q.Get()) using (o)
            {
                sockets++;
                name    = o["Name"]?.ToString()?.Trim() ?? name;
                maxMhz  = Math.Max(maxMhz, o["MaxClockSpeed"] is { } mc ? Convert.ToDouble(mc) : 0);
                cores   += o["NumberOfCores"] is { } c ? Convert.ToInt32(c) : 0;
                logical += o["NumberOfLogicalProcessors"] is { } lp ? Convert.ToInt32(lp) : 0;
                l2      += o["L2CacheSize"] is { } x2 ? Convert.ToUInt32(x2) : 0;
                l3      += o["L3CacheSize"] is { } x3 ? Convert.ToUInt32(x3) : 0;
                if (o["VirtualizationFirmwareEnabled"] is bool vb) virt = vb;
            }
            d.CpuName        = name.Length > 0 ? name : "—";
            d.BaseGhz        = maxMhz / 1000.0;
            d.Sockets        = Math.Max(1, sockets);
            d.Cores          = cores; d.Logical = logical;
            d.Virtualization = virt is { } v ? (v ? "Enabled" : "Disabled") : "—";
            d.L2 = Cache(l2); d.L3 = Cache(l3);
        }
        catch { }

        // L1 isn't on Win32_Processor — sum the Level-3 (= L1) caches from Win32_CacheMemory.
        try
        {
            uint l1 = 0;
            using var q = new ManagementObjectSearcher("SELECT Level, InstalledSize FROM Win32_CacheMemory");
            foreach (ManagementObject o in q.Get()) using (o)
                if ((o["Level"] is { } lv ? Convert.ToInt32(lv) : 0) == 3)
                    l1 += o["InstalledSize"] is { } sz ? Convert.ToUInt32(sz) : 0;
            if (l1 > 0) d.L1 = Cache(l1);
        }
        catch { }

        try
        {
            var up = TimeSpan.FromMilliseconds(Environment.TickCount64);
            d.Uptime = up.Days > 0 ? $"{up.Days}d {up.Hours}h {up.Minutes}m" : $"{up.Hours}h {up.Minutes}m {up.Seconds}s";
        }
        catch { }

        // ── Memory ──
        try
        {
            var m = new MEMORYSTATUSEX { dwLength = (uint)Marshal.SizeOf<MEMORYSTATUSEX>() };
            if (GlobalMemoryStatusEx(ref m)) d.RamTotalGb = Math.Round(m.ullTotalPhys / 1073741824.0, 1);
        }
        catch { }
        try
        {
            int used = 0, total = 0, speed = 0; string type = "";
            using (var q = new ManagementObjectSearcher("SELECT Speed, SMBIOSMemoryType FROM Win32_PhysicalMemory"))
                foreach (ManagementObject o in q.Get()) using (o)
                {
                    used++;
                    if (o["Speed"] is { } sp) speed = Math.Max(speed, Convert.ToInt32(sp));
                    if (o["SMBIOSMemoryType"] is { } t) { var mt = MemType(Convert.ToInt32(t)); if (mt.Length > 0) type = mt; }
                }
            using (var q2 = new ManagementObjectSearcher("SELECT MemoryDevices FROM Win32_PhysicalMemoryArray"))
                foreach (ManagementObject o in q2.Get()) using (o)
                    total = o["MemoryDevices"] is { } md ? Convert.ToInt32(md) : total;
            d.RamSpeed = speed;
            d.RamType  = type.Length > 0 ? type : "—";
            d.Slots    = total > 0 ? $"{used} of {total}" : used.ToString();
        }
        catch { }

        // ── Disk / Network (reuse existing collectors) ──
        try
        {
            var disk = HardwareInfo.Collect().Disks.FirstOrDefault();
            if (disk != null) { d.DiskModel = disk.Model; d.DiskType = disk.Type; d.DiskSizeGb = disk.SizeGb; }
        }
        catch { }
        try
        {
            if (NetworkInfo.GetPrimaryAdapter() is { } a) { d.NetName = a.Name; d.NetType = a.Type; d.NetIp = a.IpAddress; }
        }
        catch { }

        return d;
    }

    /// <summary>Per-logical-processor utilization %, in core order (Win32_PerfFormattedData_PerfOS_Processor).</summary>
    public static double[] CoreLoads()
    {
        try
        {
            var byIndex = new SortedDictionary<int, double>();
            using var q = new ManagementObjectSearcher(
                "SELECT Name, PercentProcessorTime FROM Win32_PerfFormattedData_PerfOS_Processor");
            foreach (ManagementObject o in q.Get()) using (o)
            {
                var name = o["Name"]?.ToString();
                if (string.IsNullOrEmpty(name) || name == "_Total") continue;
                if (int.TryParse(name, out int idx))
                    byIndex[idx] = o["PercentProcessorTime"] is { } p ? Convert.ToDouble(p) : 0;
            }
            return byIndex.Values.ToArray();
        }
        catch { return Array.Empty<double>(); }
    }

    /// <summary>Live process / thread / handle totals (a process sweep — call on a slow cadence).</summary>
    public static (int procs, int threads, int handles) ProcessCounts()
    {
        int procs = 0, threads = 0, handles = 0;
        try
        {
            foreach (var p in Process.GetProcesses())
            {
                procs++;
                try { threads += p.Threads.Count; } catch { }
                try { handles += p.HandleCount; }   catch { }
                p.Dispose();
            }
        }
        catch { }
        return (procs, threads, handles);
    }

    private static string Cache(uint kb) => kb == 0 ? "—" : kb >= 1024 ? $"{kb / 1024.0:0.#} MB" : $"{kb} KB";

    private static string MemType(int t) => t switch
    {
        20 => "DDR", 21 => "DDR2", 24 => "DDR3", 26 => "DDR4", 34 or 35 => "DDR5",
        _ => "",
    };

    [StructLayout(LayoutKind.Sequential)]
    private struct MEMORYSTATUSEX
    {
        public uint  dwLength, dwMemoryLoad;
        public ulong ullTotalPhys, ullAvailPhys, ullTotalPageFile, ullAvailPageFile,
                     ullTotalVirtual, ullAvailVirtual, ullAvailExtendedVirtual;
    }
    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool GlobalMemoryStatusEx(ref MEMORYSTATUSEX lpBuffer);
}
```
