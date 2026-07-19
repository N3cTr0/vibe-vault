---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\CrashInfo.cs
---

# PartnerTool\CrashInfo.cs

```csharp
using System.Diagnostics.Eventing.Reader;
using System.Text.RegularExpressions;

namespace PartnerTool;

public record BsodEvent(DateTime Time, string StopCode, string Detail);
public record AppFault(DateTime Time, string App, string Module);
public record AppFaultGroup(DateTime Time, string App, string Module, string Note);

/// <summary>
/// Crash history: blue-screen bugchecks (System log event 1001), application crashes
/// (Application log 1000) and application hangs (1002). Turns "3 minidumps present" into
/// "IRQL_NOT_LESS_OR_EQUAL — nvlddmkm.sys" by reading the actual stop code.
/// </summary>
public class CrashInfo
{
    public List<BsodEvent> Bsods   { get; set; } = new();
    public List<AppFault>  Crashes { get; set; } = new();
    public List<AppFault>  Hangs   { get; set; } = new();

    private static readonly Regex StopCode = new(@"0x[0-9A-Fa-f]{8,16}", RegexOptions.Compiled);

    public static CrashInfo Collect()
    {
        var c = new CrashInfo();

        // ── Bugchecks (BSOD) — System log, event 1001 from the WER bugcheck provider ──
        foreach (var (time, msg) in ReadLog("System", "*[System[(EventID=1001)]]", 40))
        {
            if (msg.IndexOf("bugcheck", StringComparison.OrdinalIgnoreCase) < 0) continue;
            var code = StopCode.Match(msg);
            c.Bsods.Add(new BsodEvent(time, code.Success ? code.Value : "Unknown", Trim(msg)));
            if (c.Bsods.Count >= 15) break;
        }

        // ── Application crashes (1000) and hangs (1002) ──
        // Event IDs alone aren't unique: other providers reuse 1000 for info messages (VMware NAT
        // Service logs its startup as 1000), which rendered as blank "— · —" crash rows. Pin the
        // provider so only real WER crash/hang events get in.
        // Read deeper than the UI shows — repeats collapse into one grouped row (see Group), so a
        // shallow read would undercount "N×" for the frequent offenders.
        foreach (var (time, msg) in ReadLog("Application", "*[System[Provider[@Name='Application Error'] and (EventID=1000)]]", 60))
        {
            c.Crashes.Add(new AppFault(time, Field(msg, "Faulting application name"), Field(msg, "Faulting module name")));
            if (c.Crashes.Count >= 60) break;
        }
        foreach (var (time, msg) in ReadLog("Application", "*[System[Provider[@Name='Application Hang'] and (EventID=1002)]]", 40))
        {
            c.Hangs.Add(new AppFault(time, Field(msg, "The program"), ""));
            if (c.Hangs.Count >= 20) break;
        }

        return c;
    }

    /// <summary>
    /// Collapse identical app+module faults into one row: latest time, a count, and — when the
    /// repeats cluster at the same time of day across 3+ days — a "daily ~HH:mm" tag, because a
    /// crash that fires on a schedule usually IS a schedule (task/scan) and that's the diagnosis.
    /// Without this the list was a wall of "wmiprvse.exe · ntdll.dll" rows drowning the one-offs.
    /// </summary>
    public static List<AppFaultGroup> Group(IEnumerable<AppFault> faults)
        => faults.GroupBy(f => (App: f.App.ToLowerInvariant(), Mod: f.Module.ToLowerInvariant()))
            .Select(g =>
            {
                var times = g.Select(f => f.Time).OrderBy(t => t).ToList();
                string note = "";
                if (times.Count > 1)
                {
                    note = $"  —  {times.Count}× since {times[0]:MM/dd/yyyy}";
                    var mins = times.Select(t => t.TimeOfDay.TotalMinutes).ToList();
                    if (times.Select(t => t.Date).Distinct().Count() >= 3 && mins.Max() - mins.Min() <= 45)
                        note += $", daily ~{TimeSpan.FromMinutes(mins.Average()):hh\\:mm}";
                }
                var latest = g.MaxBy(f => f.Time)!;
                return new AppFaultGroup(latest.Time, latest.App, latest.Module, note);
            })
            .OrderByDescending(x => x.Time).ToList();

    private static IEnumerable<(DateTime, string)> ReadLog(string log, string xpath, int max)
    {
        var list = new List<(DateTime, string)>();
        try
        {
            var query = new EventLogQuery(log, PathType.LogName, xpath) { ReverseDirection = true };
            using var reader = new EventLogReader(query);
            for (EventRecord? rec = reader.ReadEvent(); rec != null && list.Count < max; rec = reader.ReadEvent())
                using (rec)
                {
                    string msg;
                    try { msg = rec.FormatDescription() ?? ""; } catch { msg = ""; }
                    list.Add((rec.TimeCreated ?? DateTime.MinValue, msg));
                }
        }
        catch { }
        return list;
    }

    private static string Trim(string s)
    {
        s = s.Replace("\r", " ").Replace("\n", " ").Trim();
        return s.Length > 240 ? s[..240] + "…" : s;
    }

    // Pull a "Label: value" field out of an event message.
    private static string Field(string msg, string label)
    {
        var m = Regex.Match(msg, Regex.Escape(label) + @"\s*[:]?\s*([^\r\n,]+)", RegexOptions.IgnoreCase);
        return m.Success ? m.Groups[1].Value.Trim() : "—";
    }
}
```
