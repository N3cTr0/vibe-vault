---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DiagnosticsInfo.cs
---

# PartnerTool\DiagnosticsInfo.cs

```csharp
using System.Diagnostics.Eventing.Reader;
using System.IO;
using System.Management;

namespace PartnerTool;

public record EventEntry(DateTime Time, string Level, int Id, string Source, string Message);
public record DeviceProblem(string Name, string Problem);

public class DiagnosticsInfo
{
    public List<EventEntry>    Events        { get; set; } = new();
    public List<DeviceProblem> Devices       { get; set; } = new();
    public int                 MinidumpCount { get; set; }
    public DateTime?           LatestDump    { get; set; }
    public bool                MemoryDump    { get; set; }

    public static DiagnosticsInfo Collect()
    {
        var d = new DiagnosticsInfo();

        // ── Recent Critical/Error events (System + Application) ──
        var events = new List<EventEntry>();
        events.AddRange(ReadLog("System", 80));
        events.AddRange(ReadLog("Application", 80));
        d.Events = events.OrderByDescending(e => e.Time).Take(150).ToList();

        // ── Devices with driver problems ─────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, ConfigManagerErrorCode FROM Win32_PnPEntity WHERE ConfigManagerErrorCode <> 0");
            foreach (ManagementObject o in q.Get())
            {
                int code = Convert.ToInt32(o["ConfigManagerErrorCode"] ?? 0);
                var name = o["Name"]?.ToString()?.Trim();
                if (string.IsNullOrEmpty(name)) continue;
                d.Devices.Add(new DeviceProblem(name, MapDeviceCode(code)));
            }
        }
        catch { }

        // ── Crash dumps ──────────────────────────────────────
        try
        {
            const string dir = @"C:\Windows\Minidump";
            if (Directory.Exists(dir))
            {
                var files = Directory.GetFiles(dir, "*.dmp");
                d.MinidumpCount = files.Length;
                if (files.Length > 0) d.LatestDump = files.Max(File.GetLastWriteTime);
            }
            d.MemoryDump = File.Exists(@"C:\Windows\MEMORY.DMP");
        }
        catch { }

        return d;
    }

    private static IEnumerable<EventEntry> ReadLog(string log, int maxItems)
    {
        var list = new List<EventEntry>();
        try
        {
            // Level 1 = Critical, 2 = Error; within the last 10 days
            const string xpath =
                "*[System[(Level=1 or Level=2) and TimeCreated[timediff(@SystemTime) <= 864000000]]]";
            var query = new EventLogQuery(log, PathType.LogName, xpath) { ReverseDirection = true };
            using var reader = new EventLogReader(query);

            for (EventRecord? rec = reader.ReadEvent(); rec != null && list.Count < maxItems; rec = reader.ReadEvent())
            {
                using (rec)
                {
                    string level;
                    try { level = rec.LevelDisplayName ?? ""; }
                    catch { level = ""; }
                    if (string.IsNullOrEmpty(level)) level = rec.Level == 1 ? "Critical" : "Error";

                    string msg;
                    try { msg = rec.FormatDescription() ?? ""; } catch { msg = ""; }
                    msg = msg.Replace("\r", " ").Replace("\n", " ").Trim();
                    if (msg.Length > 240) msg = msg[..240] + "…";

                    list.Add(new EventEntry(
                        rec.TimeCreated ?? DateTime.MinValue,
                        level, rec.Id,
                        rec.ProviderName ?? "Unknown",
                        msg.Length == 0 ? "(no description available)" : msg));
                }
            }
        }
        catch { }
        return list;
    }

    private static string MapDeviceCode(int code) => code switch
    {
        1  => "Not configured correctly",
        3  => "Driver corrupted / low memory",
        10 => "Cannot start",
        12 => "Not enough resources",
        14 => "Needs a restart",
        18 => "Reinstall the drivers",
        19 => "Registry / configuration corrupt",
        21 => "Being removed",
        22 => "Disabled",
        24 => "Not present or not working",
        28 => "Drivers not installed",
        29 => "Disabled in firmware",
        31 => "Not working properly",
        37 => "Driver failed to initialize",
        39 => "Driver missing or corrupt",
        43 => "Stopped after reporting problems",
        45 => "Not currently connected",
        _  => $"Problem code {code}",
    };
}
```
