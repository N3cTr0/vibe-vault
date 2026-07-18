---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SmartAttributes.cs
---

# PartnerTool\SmartAttributes.cs

```csharp
using System.Management;

namespace PartnerTool;

public record SmartAttr(int Id, string Name, int Current, int Worst, int? Threshold, long Raw)
{
    public bool BelowThreshold => Threshold is { } t && t > 0 && Current <= t;
    public string Display => $"{Id} {Name}";
}
public record SmartDrive(string Instance, bool? PredictFailure, List<SmartAttr> Attributes);

/// <summary>
/// Full SMART attribute table (CrystalDiskInfo-style) for ATA/SATA drives, read from the
/// root\wmi MSStorageDriver_* classes and parsed from the raw 512-byte SMART structures.
/// NVMe drives generally don't expose this — their temperature/wear is already covered by
/// the Storage reliability counters on the System Info page.
/// </summary>
public static class SmartAttributes
{
    private static readonly Dictionary<int, string> Names = new()
    {
        [1]="Read Error Rate", [3]="Spin-Up Time", [4]="Start/Stop Count", [5]="Reallocated Sectors",
        [7]="Seek Error Rate", [9]="Power-On Hours", [10]="Spin Retry Count", [12]="Power Cycle Count",
        [173]="Wear Leveling Count", [174]="Unexpected Power Loss", [177]="Wear Range Delta",
        [184]="End-to-End Error", [187]="Reported Uncorrectable", [188]="Command Timeout",
        [190]="Airflow Temperature", [194]="Temperature", [196]="Reallocation Events",
        [197]="Current Pending Sectors", [198]="Offline Uncorrectable", [199]="UDMA CRC Errors",
        [231]="SSD Life Left", [233]="Media Wearout", [241]="Total LBAs Written", [242]="Total LBAs Read",
    };

    public static List<SmartDrive> Collect()
    {
        var thresholds = new Dictionary<string, Dictionary<int, int>>();
        var failure    = new Dictionary<string, bool>();
        var drives     = new List<SmartDrive>();

        // Thresholds (per instance)
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT InstanceName, VendorSpecific FROM MSStorageDriver_FailurePredictThresholds");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var inst = o["InstanceName"]?.ToString() ?? "";
                if (o["VendorSpecific"] is byte[] raw) thresholds[inst] = ParseThresholds(raw);
            }
        }
        catch { }

        // Predict-failure flag
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT InstanceName, PredictFailure FROM MSStorageDriver_FailurePredictStatus");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var inst = o["InstanceName"]?.ToString() ?? "";
                failure[inst] = o["PredictFailure"] is bool b && b;
            }
        }
        catch { }

        // Attribute data
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT InstanceName, VendorSpecific FROM MSStorageDriver_FailurePredictData");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var inst = o["InstanceName"]?.ToString() ?? "";
                if (o["VendorSpecific"] is not byte[] raw) continue;
                var thr  = thresholds.TryGetValue(inst, out var t) ? t : new();
                var attrs = ParseData(raw, thr);
                if (attrs.Count > 0)
                    drives.Add(new SmartDrive(inst, failure.TryGetValue(inst, out var f) ? f : null, attrs));
            }
        }
        catch { }

        return drives;
    }

    // SMART data: 2-byte revision header, then 30 × 12-byte attribute entries.
    // entry: [0]=id [1..2]=flags [3]=current [4]=worst [5..10]=raw(6, little-endian) [11]=reserved
    private static List<SmartAttr> ParseData(byte[] b, Dictionary<int, int> thresholds)
    {
        var list = new List<SmartAttr>();
        try
        {
            for (int off = 2; off + 12 <= b.Length; off += 12)
            {
                int id = b[off];
                if (id == 0) continue;
                int cur = b[off + 3], worst = b[off + 4];
                long raw = 0;
                for (int i = 0; i < 6; i++) raw |= (long)b[off + 5 + i] << (8 * i);
                int? thr = thresholds.TryGetValue(id, out var tv) ? tv : null;
                list.Add(new SmartAttr(id, Names.TryGetValue(id, out var n) ? n : $"Attribute {id}", cur, worst, thr, raw));
            }
        }
        catch { }
        return list;
    }

    // Thresholds: 2-byte header, then 30 × 12-byte entries: [0]=id [1]=threshold, rest reserved.
    private static Dictionary<int, int> ParseThresholds(byte[] b)
    {
        var map = new Dictionary<int, int>();
        try
        {
            for (int off = 2; off + 12 <= b.Length; off += 12)
            {
                int id = b[off];
                if (id == 0) continue;
                map[id] = b[off + 1];
            }
        }
        catch { }
        return map;
    }
}
```
