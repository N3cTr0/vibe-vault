---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PowerEventsInfo.cs
---

# PartnerTool\PowerEventsInfo.cs

```csharp
using System.Diagnostics.Eventing.Reader;

namespace PartnerTool;

public record PowerEvent(DateTime Time, string Kind, string Detail);

/// <summary>
/// Boot / shutdown / unexpected-restart history from the System event log — the quick
/// answer to "why did it reboot?". Covers clean shutdowns (6006), unexpected shutdowns
/// (6008), dirty power loss (Kernel-Power 41) and user/process-initiated restarts (1074).
/// </summary>
public static class PowerEventsInfo
{
    public static List<PowerEvent> Collect(int days = 14, int max = 30)
    {
        var list = new List<PowerEvent>();
        try
        {
            long window = (long)days * 24 * 60 * 60 * 1000;
            string xpath =
                "*[System[(EventID=41 or EventID=1074 or EventID=6005 or EventID=6006 or EventID=6008) " +
                $"and TimeCreated[timediff(@SystemTime) <= {window}]]]";
            var query = new EventLogQuery("System", PathType.LogName, xpath) { ReverseDirection = true };
            using var reader = new EventLogReader(query);

            for (EventRecord? rec = reader.ReadEvent(); rec != null && list.Count < max; rec = reader.ReadEvent())
            {
                using (rec)
                {
                    var (kind, fallback) = rec.Id switch
                    {
                        41   => ("Unexpected shutdown", "System rebooted without a clean shutdown (power loss or crash)."),
                        1074 => ("Restart / shutdown", "A user or process initiated a shutdown or restart."),
                        6005 => ("System started", "The event log service started (system boot)."),
                        6006 => ("Clean shutdown", "The event log service stopped (clean shutdown)."),
                        6008 => ("Unexpected shutdown", "The previous shutdown was unexpected."),
                        _    => ("Power event", ""),
                    };
                    string detail;
                    try { detail = rec.FormatDescription() ?? fallback; }
                    catch { detail = fallback; }
                    detail = detail.Replace("\r", " ").Replace("\n", " ").Trim();
                    if (detail.Length > 220) detail = detail[..220] + "…";

                    list.Add(new PowerEvent(rec.TimeCreated ?? DateTime.MinValue, kind,
                        detail.Length == 0 ? fallback : detail));
                }
            }
        }
        catch { }
        return list;
    }
}
```
