---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TaskManagerInfo.cs
---

# PartnerTool\TaskManagerInfo.cs

```csharp
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace PartnerTool;

/// <summary>One row of the Processes page — the fields Windows Task Manager shows that we can
/// read without ETW. (Per-process Network needs an ETW session, so that column is omitted.)</summary>
public class ProcRow
{
    public string Name    { get; init; } = "";
    public int    Pid     { get; init; }
    public string Status  { get; init; } = "Running";
    public string User    { get; init; } = "";
    public double Cpu     { get; init; }             // percent, 0-100 (all cores)
    public double MemMb   { get; init; }             // private bytes
    public double DiskMbs { get; init; }             // read+write rate since last sample
    public int    Threads { get; init; }
    public int    Handles { get; init; }
    public string Arch    { get; init; } = "";
    public string Desc    { get; init; } = "";
    public string Path    { get; init; } = "";
    public bool   Critical { get; init; }            // OS marks it critical — killing bugchecks Windows

    public string CpuText  => Cpu  >= 0.05 ? $"{Cpu:F1}%" : "0%";
    public string MemText  => MemMb >= 1024 ? $"{MemMb / 1024:F1} GB" : $"{MemMb:F0} MB";
    public string DiskText => DiskMbs >= 0.05 ? $"{DiskMbs:F1} MB/s" : "0 MB/s";
    public string PathTip  => Path.Length > 0 ? Path : Name;

    public bool CanEnd => !ProcessInfo.IsProtected(Pid, Name) && !Critical;
}

/// <summary>
/// Stateful process sampler: CPU %% and disk rate are deltas between consecutive
/// <see cref="Sample"/> calls, so keep one instance alive per page. Expensive per-process
/// lookups (user, description, architecture) are cached by PID+start-ticks.
/// </summary>
public class ProcessSampler
{
    private record Prev(TimeSpan Cpu, ulong IoBytes, DateTime At);
    private record Static(string User, string Desc, string Arch, string Path, bool Critical, long StartTicks);

    private readonly Dictionary<int, Prev>   _prev   = new();
    private readonly Dictionary<int, Static> _static = new();

    /// <summary>Snapshot all processes. Call on a background thread — it touches every process.</summary>
    public List<ProcRow> Sample()
    {
        var now  = DateTime.UtcNow;
        var rows = new List<ProcRow>(256);
        var seen = new HashSet<int>();
        int cores = Environment.ProcessorCount;

        foreach (var p in Process.GetProcesses())
        {
            try
            {
                if (p.Id == 0) continue;   // Idle pseudo-process — taskmgr hides the real one too
                seen.Add(p.Id);

                // CPU/IO deltas vs the previous sample (0 on the first sighting).
                TimeSpan cpuNow = TimeSpan.Zero;
                try { cpuNow = p.TotalProcessorTime; } catch { /* access denied (protected) */ }
                ulong ioNow = ReadIoBytes(p.Id);

                double cpuPct = 0, diskMbs = 0;
                if (_prev.TryGetValue(p.Id, out var prev))
                {
                    var elapsed = (now - prev.At).TotalSeconds;
                    if (elapsed > 0.2)
                    {
                        cpuPct  = Math.Clamp((cpuNow - prev.Cpu).TotalSeconds / elapsed / cores * 100, 0, 100);
                        if (ioNow >= prev.IoBytes)
                            diskMbs = (ioNow - prev.IoBytes) / elapsed / 1048576.0;
                    }
                }
                _prev[p.Id] = new Prev(cpuNow, ioNow, now);

                var st = GetStatic(p);

                string status = "Running";
                try
                {
                    if (p.MainWindowHandle != IntPtr.Zero && !p.Responding) status = "Not responding";
                }
                catch { }

                double memMb = 0;
                try { memMb = p.PrivateMemorySize64 / 1048576.0; } catch { }
                int threads = 0, handles = 0;
                try { threads = p.Threads.Count; } catch { }
                try { handles = p.HandleCount; }  catch { }

                rows.Add(new ProcRow
                {
                    Name = p.ProcessName, Pid = p.Id, Status = status, User = st.User,
                    Cpu = cpuPct, MemMb = memMb, DiskMbs = diskMbs,
                    Threads = threads, Handles = handles,
                    Arch = st.Arch, Desc = st.Desc, Path = st.Path, Critical = st.Critical,
                });
            }
            catch { /* exited mid-walk — skip */ }
            finally { p.Dispose(); }
        }

        // Drop dead PIDs from the caches so a recycled PID can't inherit stale numbers.
        foreach (var pid in _prev.Keys.Where(k => !seen.Contains(k)).ToList())  _prev.Remove(pid);
        foreach (var pid in _static.Keys.Where(k => !seen.Contains(k)).ToList()) _static.Remove(pid);
        return rows;
    }

    // ── static per-process facts (cached — PID + start time guards against PID reuse) ──

    private Static GetStatic(Process p)
    {
        long startTicks = 0;
        try { startTicks = p.StartTime.Ticks; } catch { }
        if (_static.TryGetValue(p.Id, out var s) && s.StartTicks == startTicks) return s;

        string user = LookupUser(p.Id);
        string path = "", desc = "";
        try
        {
            path = p.MainModule?.FileName ?? "";
            if (path.Length > 0)
                desc = FileVersionInfo.GetVersionInfo(path).FileDescription ?? "";
        }
        catch { /* protected/system process */ }
        var made = new Static(user, desc, LookupArch(p.Id), path, ProcessInfo.IsOsCritical(p.Id), startTicks);
        _static[p.Id] = made;
        return made;
    }

    private static string LookupUser(int pid)
    {
        IntPtr hProc = IntPtr.Zero, hTok = IntPtr.Zero, buf = IntPtr.Zero;
        try
        {
            hProc = OpenProcess(0x1000 /*QUERY_LIMITED_INFORMATION*/, false, pid);
            if (hProc == IntPtr.Zero) return "";
            if (!OpenProcessToken(hProc, 0x0008 /*TOKEN_QUERY*/, out hTok)) return "";

            GetTokenInformation(hTok, 1 /*TokenUser*/, IntPtr.Zero, 0, out int len);
            if (len == 0) return "";
            buf = Marshal.AllocHGlobal(len);
            if (!GetTokenInformation(hTok, 1, buf, len, out _)) return "";

            var sid = Marshal.ReadIntPtr(buf);   // TOKEN_USER.User.Sid
            var name = new System.Text.StringBuilder(64);
            var dom  = new System.Text.StringBuilder(64);
            int nLen = name.Capacity, dLen = dom.Capacity;
            if (!LookupAccountSid(null, sid, name, ref nLen, dom, ref dLen, out _)) return "";
            // Bare name like taskmgr's Details tab (it drops the machine/domain prefix).
            return name.ToString();
        }
        catch { return ""; }
        finally
        {
            if (buf   != IntPtr.Zero) Marshal.FreeHGlobal(buf);
            if (hTok  != IntPtr.Zero) CloseHandle(hTok);
            if (hProc != IntPtr.Zero) CloseHandle(hProc);
        }
    }

    private static string LookupArch(int pid)
    {
        IntPtr h = IntPtr.Zero;
        try
        {
            h = OpenProcess(0x1000, false, pid);
            if (h == IntPtr.Zero) return "";
            if (IsWow64Process2(h, out ushort procMachine, out _))
                return procMachine switch
                {
                    0        => Environment.Is64BitOperatingSystem ? "x64" : "x86",   // not WOW64 → native
                    0x014C   => "x86",
                    0x01C4   => "ARM32",
                    _        => "?",
                };
            return "";
        }
        catch { return ""; }
        finally { if (h != IntPtr.Zero) CloseHandle(h); }
    }

    private static ulong ReadIoBytes(int pid)
    {
        IntPtr h = IntPtr.Zero;
        try
        {
            h = OpenProcess(0x1000, false, pid);
            if (h == IntPtr.Zero) return 0;
            return GetProcessIoCounters(h, out var io) ? io.ReadTransferCount + io.WriteTransferCount + io.OtherTransferCount : 0;
        }
        catch { return 0; }
        finally { if (h != IntPtr.Zero) CloseHandle(h); }
    }

    // ── P/Invoke ──────────────────────────────────────────────────────────

    [StructLayout(LayoutKind.Sequential)]
    private struct IO_COUNTERS
    {
        public ulong ReadOperationCount, WriteOperationCount, OtherOperationCount;
        public ulong ReadTransferCount, WriteTransferCount, OtherTransferCount;
    }

    [DllImport("kernel32.dll")] private static extern IntPtr OpenProcess(int access, bool inherit, int pid);
    [DllImport("kernel32.dll")] private static extern bool CloseHandle(IntPtr h);
    [DllImport("kernel32.dll")] private static extern bool GetProcessIoCounters(IntPtr h, out IO_COUNTERS counters);
    [DllImport("kernel32.dll")] private static extern bool IsWow64Process2(IntPtr h, out ushort processMachine, out ushort nativeMachine);
    [DllImport("advapi32.dll")] private static extern bool OpenProcessToken(IntPtr h, uint access, out IntPtr token);
    [DllImport("advapi32.dll")] private static extern bool GetTokenInformation(IntPtr token, int cls, IntPtr info, int len, out int retLen);
    [DllImport("advapi32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    private static extern bool LookupAccountSid(string? system, IntPtr sid,
        System.Text.StringBuilder name, ref int nameLen, System.Text.StringBuilder domain, ref int domLen, out int use);
}
```
