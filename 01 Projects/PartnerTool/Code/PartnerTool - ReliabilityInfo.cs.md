---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ReliabilityInfo.cs
---

# PartnerTool\ReliabilityInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record ReliabilityRecord(DateTime Time, string Source, string Message);

/// <summary>
/// Windows Reliability Monitor data (same as perfmon /rel). Split deliberately into two
/// calls because the two WMI classes have wildly different costs:
///   • <see cref="CollectIndex"/> reads Win32_ReliabilityStabilityMetrics — a couple of
///     seconds — and gives the 1–10 System Stability Index. Safe for the startup snapshot.
///   • <see cref="CollectRecords"/> reads Win32_ReliabilityRecords, which can take the
///     better part of a MINUTE on some machines (the provider re-aggregates the whole
///     reliability database). Never call it on the startup path — load it on demand.
/// </summary>
public class ReliabilityInfo
{
    /// <summary>Latest System Stability Index 0–10 (10 = perfectly stable), or null if unavailable.</summary>
    public double? StabilityIndex { get; set; }
    /// <summary>Best (highest) index seen in the last 7 days — context for a low current value, which
    /// Windows drops for app installs / updates / driver changes, not just crashes.</summary>
    public double? RecentPeak { get; set; }
    public List<ReliabilityRecord> Records { get; set; } = new();

    /// <summary>Fast: stability index only (~2s). Used by the startup snapshot.</summary>
    public static ReliabilityInfo CollectIndex()
    {
        var r = new ReliabilityInfo();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT SystemStabilityIndex, TimeGenerated FROM Win32_ReliabilityStabilityMetrics");
            DateTime newest = DateTime.MinValue;
            var weekAgo = DateTime.Now.AddDays(-7);
            foreach (ManagementObject o in q.Get())
            {
                if (o["SystemStabilityIndex"] is not { } idxObj) continue;
                double idx = Convert.ToDouble(idxObj);
                var t = CimDate(o["TimeGenerated"]?.ToString());
                if (t >= newest) { newest = t; r.StabilityIndex = idx; }          // current = most recent record
                if (t >= weekAgo && idx > (r.RecentPeak ?? 0)) r.RecentPeak = idx; // best in the last week
            }
        }
        catch { }
        return r;
    }

    /// <summary>Slow (can take ~minute): the detailed reliability records. On-demand only.</summary>
    public static List<ReliabilityRecord> CollectRecords()
    {
        var list = new List<ReliabilityRecord>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT SourceName, ProductName, Message, TimeGenerated FROM Win32_ReliabilityRecords");
            foreach (ManagementObject o in q.Get())
            {
                var msg = o["Message"]?.ToString()?.Replace("\r", " ").Replace("\n", " ").Trim();
                if (string.IsNullOrWhiteSpace(msg))
                    msg = o["ProductName"]?.ToString()?.Trim() ?? "";
                if (msg.Length > 200) msg = msg[..200] + "…";
                list.Add(new ReliabilityRecord(
                    CimDate(o["TimeGenerated"]?.ToString()),
                    o["SourceName"]?.ToString()?.Trim() is { Length: > 0 } s ? s : "System",
                    msg));
            }
        }
        catch { }
        return list.OrderByDescending(x => x.Time).Take(40).ToList();
    }

    private static DateTime CimDate(string? cim)
    {
        if (string.IsNullOrEmpty(cim)) return DateTime.MinValue;
        try { return ManagementDateTimeConverter.ToDateTime(cim); }
        catch { return DateTime.MinValue; }
    }
}
```
