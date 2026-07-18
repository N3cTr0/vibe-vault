---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BootPerfInfo.cs
---

# PartnerTool\BootPerfInfo.cs

```csharp
using System.Diagnostics.Eventing.Reader;
using System.Xml.Linq;

namespace PartnerTool;

public record BootTime(DateTime When, double Seconds);
public record BootContributor(string Kind, string Name, double Seconds);

/// <summary>
/// Boot-performance data from the Microsoft-Windows-Diagnostics-Performance/Operational
/// log — Windows already times every boot and names the apps/drivers/services that slow it
/// down. Event 100 = boot duration; 101/102/103 = slow app/driver/service contributors.
/// </summary>
public class BootPerfInfo
{
    public List<BootTime>        BootTimes    { get; set; } = new();
    public List<BootContributor> Contributors { get; set; } = new();
    public double? LatestBootSeconds => BootTimes.Count > 0 ? BootTimes[0].Seconds : null;

    private const string Log = "Microsoft-Windows-Diagnostics-Performance/Operational";
    private static readonly XNamespace Ns = "http://schemas.microsoft.com/win/2004/08/events/event";

    public static BootPerfInfo Collect()
    {
        var info = new BootPerfInfo();
        try
        {
            // Event 100 = boot; 101 app, 102 driver, 103 service slowed boot.
            var query = new EventLogQuery(Log, PathType.LogName,
                "*[System[(EventID=100 or EventID=101 or EventID=102 or EventID=103)]]")
            { ReverseDirection = true };
            using var reader = new EventLogReader(query);

            for (EventRecord? rec = reader.ReadEvent(); rec != null; rec = reader.ReadEvent())
                using (rec)
                {
                    XElement? data;
                    try { data = XElement.Parse(rec.ToXml()).Element(Ns + "EventData"); }
                    catch { continue; }
                    if (data == null) continue;

                    string D(string name) => data.Elements(Ns + "Data")
                        .FirstOrDefault(e => (string?)e.Attribute("Name") == name)?.Value ?? "";

                    if (rec.Id == 100)
                    {
                        if (info.BootTimes.Count < 10 && double.TryParse(D("BootTime"), out var ms))
                            info.BootTimes.Add(new BootTime(rec.TimeCreated ?? DateTime.MinValue, Math.Round(ms / 1000.0, 1)));
                    }
                    else if (info.Contributors.Count < 25)
                    {
                        string kind = rec.Id switch { 101 => "App", 102 => "Driver", 103 => "Service", _ => "Item" };
                        var name = D("Name");
                        double.TryParse(D("TotalTime"), out var t);
                        if (t == 0) double.TryParse(D("Time"), out t);
                        if (!string.IsNullOrWhiteSpace(name))
                            info.Contributors.Add(new BootContributor(kind, name.Trim(), Math.Round(t / 1000.0, 1)));
                    }

                    if (info.BootTimes.Count >= 10 && info.Contributors.Count >= 25) break;
                }

            info.Contributors = info.Contributors.OrderByDescending(c => c.Seconds).Take(20).ToList();
        }
        catch { /* log disabled or inaccessible */ }
        return info;
    }
}
```
