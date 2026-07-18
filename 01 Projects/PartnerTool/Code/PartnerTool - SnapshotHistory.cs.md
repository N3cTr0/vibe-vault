---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SnapshotHistory.cs
---

# PartnerTool\SnapshotHistory.cs

```csharp
using System.IO;
using System.Text.Json;

namespace PartnerTool;

/// <summary>One launch's worth of headline metrics, persisted so techs can see trends.</summary>
public record HistoryEntry(
    DateTime When, double RamPct, double DiskFreeGb, double DiskPct,
    int? BatteryWearPct, double? StabilityIndex, bool RebootPending);

/// <summary>
/// Appends a compact metrics row to history.json (next to settings.json) on each launch,
/// so the tool can show whether the disk is filling up, the battery is degrading or the
/// machine is getting less stable over time. Capped to the most recent entries.
/// </summary>
public static class SnapshotHistory
{
    private const int Max = 90;
    private static readonly string FilePath =
        Path.Combine(AppContext.BaseDirectory, "history.json");

    public static List<HistoryEntry> Load()
    {
        try
        {
            if (File.Exists(FilePath))
                return JsonSerializer.Deserialize<List<HistoryEntry>>(File.ReadAllText(FilePath)) ?? new();
        }
        catch { }
        return new();
    }

    public static void Append(SystemSnapshot snap)
    {
        try
        {
            var list = Load();
            // At most one entry per day so the trend doesn't drown in same-day relaunches.
            list.RemoveAll(e => e.When.Date == DateTime.Now.Date);
            list.Add(new HistoryEntry(
                snap.CapturedAt,
                Math.Round(snap.Perf.RamPct, 0),
                Math.Round(snap.Perf.DiskTotalGb - snap.Perf.DiskUsedGb, 1),
                Math.Round(snap.Perf.DiskPct, 0),
                snap.Hardware.BatteryWearPct,
                snap.Reliability?.StabilityIndex is { } si ? Math.Round(si, 1) : null,
                snap.RebootPending));
            if (list.Count > Max) list = list.OrderByDescending(e => e.When).Take(Max).OrderBy(e => e.When).ToList();
            File.WriteAllText(FilePath,
                JsonSerializer.Serialize(list, new JsonSerializerOptions { WriteIndented = true }));
        }
        catch { /* read-only location — history is best-effort */ }
    }
}
```
