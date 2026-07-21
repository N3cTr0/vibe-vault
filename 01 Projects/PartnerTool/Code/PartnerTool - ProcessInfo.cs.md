---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ProcessInfo.cs
---

# PartnerTool\ProcessInfo.cs

```csharp
using System.Diagnostics;

namespace PartnerTool;

public record ProcInfo(int Pid, string Name, double CpuPct, double MemMb)
{
    /// <summary>
    /// False for rows the tool refuses to end — critical Windows processes (which would BSOD) plus
    /// the tool's own process — so the End button can disable itself instead of failing on click.
    /// Checks both our name list and the OS's own critical flag.
    /// </summary>
    public bool CanEnd => !ProcessInfo.IsProtected(Pid, Name) && !ProcessInfo.IsOsCritical(Pid);
}

/// <summary>
/// A lightweight "top processes" snapshot — like the Task Manager Processes tab. CPU% is
/// measured by sampling each process's total processor time across a short interval and
/// normalising by the logical-processor count, so it's a real over-the-window figure.
/// </summary>
public static class ProcessInfo
{
    public static async Task<List<ProcInfo>> TopAsync(int count = 14)
    {
        int cpuCount = Math.Max(1, Environment.ProcessorCount);
        var firstCpu = SampleCpu();            // pid -> processor time at T0
        await Task.Delay(500);
        const double elapsedMs = 500.0;

        var rows = new List<ProcInfo>();
        foreach (var (pid, name, cpu2, mem) in Sample())
        {
            double cpuPct = 0;
            if (firstCpu.TryGetValue(pid, out var prev))
                cpuPct = Math.Clamp((cpu2 - prev).TotalMilliseconds / (elapsedMs * cpuCount) * 100.0, 0, 100);
            rows.Add(new ProcInfo(pid, name, cpuPct, mem));
        }

        // Most interesting first: highest CPU, then biggest memory.
        return rows
            .OrderByDescending(p => p.CpuPct)
            .ThenByDescending(p => p.MemMb)
            .Take(count)
            .ToList();
    }

    private static Dictionary<int, TimeSpan> SampleCpu()
    {
        var map = new Dictionary<int, TimeSpan>();
        foreach (var (pid, _, cpu, _) in Sample())
            map[pid] = cpu;
        return map;
    }

    private static List<(int Pid, string Name, TimeSpan Cpu, double MemMb)> Sample()
    {
        var list = new List<(int, string, TimeSpan, double)>();
        foreach (var p in Process.GetProcesses())
        {
            try
            {
                if (p.Id == 0) continue;
                list.Add((p.Id, p.ProcessName, p.TotalProcessorTime, p.WorkingSet64 / 1048576.0));
            }
            catch { /* access denied / exited — skip */ }
            finally { p.Dispose(); }
        }
        return list;
    }

    /// <summary>
    /// Processes that must never be killed. The OS ones trigger a Windows bugcheck
    /// (CRITICAL_PROCESS_DIED / 0xEF) and an instant BSOD; the AV/EDR ones would drop the
    /// machine's protection (or at best throw access-denied via tamper protection). Blocked outright.
    /// </summary>
    private static readonly HashSet<string> Critical = new(StringComparer.OrdinalIgnoreCase)
    {
        "system", "idle", "registry", "memory compression", "secure system",
        "smss", "csrss", "wininit", "winlogon", "services", "lsass", "lsaiso", "fontdrvhost",
        // Killing the wrong svchost (e.g. the RPC/DcomLaunch host) also bugchecks — and the tool
        // can't tell which services an instance hosts, so block them all. Use services.msc instead.
        "svchost",
        // Security / endpoint protection engines — never kill AV/EDR from a support tool. Mirrors
        // the guards that already block stopping their services (ServicesInfo.Critical) and
        // disabling their startup entries (StartupInfo.IsSecurityCritical).
        "MsMpEng", "NisSrv", "MpDefenderCoreService", "SecurityHealthService", "MsSense",
        "SentinelAgent", "SentinelHelperService", "SentinelServiceHost", "SentinelStaticEngine",
        "HuntressAgent", "HuntressRio", "CSFalconService", "CSFalconContainer", "CylanceSvc",
    };

    public static bool IsCritical(string name) => Critical.Contains(name);

    /// <summary>True for processes the tool must not end: critical Windows procs, or the tool itself.</summary>
    public static bool IsProtected(int pid, string name) => pid == Environment.ProcessId || IsCritical(name);

    /// <summary>
    /// Ask Windows whether killing this process would bugcheck the machine (CRITICAL_PROCESS_DIED).
    /// This is the authoritative check — it catches critical processes that aren't in our name list,
    /// including third-party ones the OS marks critical. Returns false if it can't be determined
    /// (access denied / already exited) — the name list remains the backstop.
    /// </summary>
    public static bool IsOsCritical(int pid)
    {
        if (pid <= 4) return true;   // System / Idle — never openable, always fatal
        IntPtr h = OpenProcess(0x1000 /*QUERY_LIMITED_INFORMATION*/, false, pid);
        if (h == IntPtr.Zero) return false;
        try { return IsProcessCritical(h, out bool critical) && critical; }
        finally { CloseHandle(h); }
    }

    public static bool TryKill(int pid)
    {
        try
        {
            using var p = Process.GetProcessById(pid);
            // Three gates, any of which blocks the kill: our name list (AV/EDR + known-fatal),
            // the tool's own PID, and the OS's authoritative "killing this bugchecks Windows" flag.
            if (IsProtected(pid, p.ProcessName)) return false;
            if (IsOsCritical(pid)) return false;
            p.Kill();
            return true;
        }
        catch { return false; }
    }

    [System.Runtime.InteropServices.DllImport("kernel32.dll")]
    private static extern IntPtr OpenProcess(int access, bool inheritHandle, int pid);
    [System.Runtime.InteropServices.DllImport("kernel32.dll")]
    private static extern bool CloseHandle(IntPtr h);
    [System.Runtime.InteropServices.DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool IsProcessCritical(IntPtr hProcess, out bool critical);
}
```
