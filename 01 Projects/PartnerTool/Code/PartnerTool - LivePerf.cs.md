---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\LivePerf.cs
---

# PartnerTool\LivePerf.cs

```csharp
using System.Net.NetworkInformation;
using System.Runtime.InteropServices;

namespace PartnerTool;

public record LiveSample(double CpuPct, double RamPct, double RamUsedGb, double RamTotalGb,
                         double DiskPct, double NetDownMbps, double NetUpMbps);

/// <summary>
/// Lightweight live performance sampler for the Monitor page. CPU and RAM come from
/// kernel APIs (GetSystemTimes / GlobalMemoryStatusEx — no perf-counter dependency, no
/// flakiness); disk-busy % from a WMI performance class; network throughput from network
/// interface byte deltas. Stateful: keep one instance and call <see cref="Sample"/> on a timer.
/// </summary>
public sealed class LivePerf
{
    private long _prevIdle, _prevKernel, _prevUser;
    private long _prevRx, _prevTx;
    private DateTime _prevTime;
    private bool _primed;

    public LiveSample Sample()
    {
        double cpu  = SampleCpu();
        var (ramPct, usedGb, totalGb) = SampleRam();
        double disk = SampleDisk();
        var (down, up) = SampleNet();
        return new LiveSample(cpu, ramPct, usedGb, totalGb, disk, down, up);
    }

    // ── CPU ──────────────────────────────────────────────────
    private double SampleCpu()
    {
        try
        {
            if (!GetSystemTimes(out var idle, out var kernel, out var user)) return 0;
            long i = ToLong(idle), k = ToLong(kernel), u = ToLong(user);
            double pct = 0;
            if (_primed)
            {
                long dIdle = i - _prevIdle, dKernel = k - _prevKernel, dUser = u - _prevUser;
                long total = dKernel + dUser;              // kernel time already includes idle
                if (total > 0) pct = Math.Clamp((total - dIdle) * 100.0 / total, 0, 100);
            }
            _prevIdle = i; _prevKernel = k; _prevUser = u; _primed = true;
            return pct;
        }
        catch { return 0; }
    }

    // ── RAM ──────────────────────────────────────────────────
    private static (double pct, double usedGb, double totalGb) SampleRam()
    {
        try
        {
            var m = new MEMORYSTATUSEX { dwLength = (uint)Marshal.SizeOf<MEMORYSTATUSEX>() };
            if (GlobalMemoryStatusEx(ref m))
            {
                double total = m.ullTotalPhys / 1073741824.0;
                double avail = m.ullAvailPhys / 1073741824.0;
                double used  = total - avail;
                return (total > 0 ? used / total * 100 : 0, used, total);
            }
        }
        catch { }
        return (0, 0, 0);
    }

    // ── Disk busy % ──────────────────────────────────────────
    private static double SampleDisk()
    {
        try
        {
            using var q = new System.Management.ManagementObjectSearcher(
                "SELECT PercentDiskTime FROM Win32_PerfFormattedData_PerfDisk_PhysicalDisk WHERE Name='_Total'");
            foreach (System.Management.ManagementObject o in q.Get())
            {
                var v = Convert.ToDouble(o["PercentDiskTime"] ?? 0);
                return Math.Clamp(v, 0, 100);
            }
        }
        catch { }
        return 0;
    }

    // ── Network throughput ───────────────────────────────────
    private (double down, double up) SampleNet()
    {
        try
        {
            long rx = 0, tx = 0;
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
            {
                if (ni.OperationalStatus != OperationalStatus.Up) continue;
                if (ni.NetworkInterfaceType == NetworkInterfaceType.Loopback) continue;
                try { var s = ni.GetIPStatistics(); rx += s.BytesReceived; tx += s.BytesSent; } catch { }
            }
            var now = DateTime.UtcNow;
            double down = 0, up = 0;
            if (_prevTime != default)
            {
                double secs = (now - _prevTime).TotalSeconds;
                if (secs > 0)
                {
                    down = Math.Max(0, (rx - _prevRx)) * 8 / 1_000_000.0 / secs;   // Mbps
                    up   = Math.Max(0, (tx - _prevTx)) * 8 / 1_000_000.0 / secs;
                }
            }
            _prevRx = rx; _prevTx = tx; _prevTime = now;
            return (Math.Round(down, 1), Math.Round(up, 1));
        }
        catch { return (0, 0); }
    }

    private static long ToLong(FILETIME ft) => ((long)ft.dwHighDateTime << 32) | (uint)ft.dwLowDateTime;

    [StructLayout(LayoutKind.Sequential)]
    private struct FILETIME { public uint dwLowDateTime; public int dwHighDateTime; }

    [StructLayout(LayoutKind.Sequential)]
    private struct MEMORYSTATUSEX
    {
        public uint dwLength;
        public uint dwMemoryLoad;
        public ulong ullTotalPhys, ullAvailPhys, ullTotalPageFile, ullAvailPageFile,
                     ullTotalVirtual, ullAvailVirtual, ullAvailExtendedVirtual;
    }

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool GetSystemTimes(out FILETIME idle, out FILETIME kernel, out FILETIME user);

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool GlobalMemoryStatusEx(ref MEMORYSTATUSEX lpBuffer);
}
```
