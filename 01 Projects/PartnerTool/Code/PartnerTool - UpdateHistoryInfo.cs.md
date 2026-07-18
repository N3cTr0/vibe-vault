---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\UpdateHistoryInfo.cs
---

# PartnerTool\UpdateHistoryInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record UpdateEntry(DateTime Date, string Title, string Result);

/// <summary>
/// Recent Windows Update history — "when was this machine last patched?". Primary source
/// is the Windows Update Agent COM API (Microsoft.Update.Session), which includes quality
/// and feature updates with install dates and result codes. Falls back to the installed
/// hotfix list (Win32_QuickFixEngineering) if the COM API is unavailable.
/// </summary>
public class UpdateHistoryInfo
{
    public DateTime?         LastInstalled { get; set; }
    public List<UpdateEntry> Recent        { get; set; } = new();

    public static UpdateHistoryInfo Collect()
    {
        var info = new UpdateHistoryInfo();

        try
        {
            var t = Type.GetTypeFromProgID("Microsoft.Update.Session");
            if (t != null)
            {
                dynamic session  = Activator.CreateInstance(t)!;
                dynamic searcher = session.CreateUpdateSearcher();
                int total = searcher.GetTotalHistoryCount();
                if (total > 0)
                {
                    dynamic history = searcher.QueryHistory(0, Math.Min(total, 40));
                    int n = history.Count;
                    var rows = new List<UpdateEntry>();
                    for (int i = 0; i < n; i++)
                    {
                        try
                        {
                            dynamic h = history.Item(i);
                            // Operation 1 = installation; skip uninstalls/other.
                            if ((int)h.Operation != 1) continue;
                            DateTime date = ((DateTime)h.Date).ToLocalTime();
                            string title  = (string)(h.Title ?? "");
                            string result = (int)h.ResultCode switch
                            {
                                2 => "Succeeded", 3 => "Succeeded (with errors)",
                                4 => "Failed",    5 => "Canceled", _ => "Pending",
                            };
                            if (!string.IsNullOrWhiteSpace(title))
                                rows.Add(new UpdateEntry(date, title.Trim(), result));
                        }
                        catch { }
                    }
                    info.Recent = rows.OrderByDescending(r => r.Date).Take(30).ToList();
                    info.LastInstalled = info.Recent.FirstOrDefault()?.Date;
                }
            }
        }
        catch { }

        if (info.Recent.Count == 0)
        {
            // Fallback: installed hotfixes
            try
            {
                using var q = new ManagementObjectSearcher(
                    "SELECT HotFixID, Description, InstalledOn FROM Win32_QuickFixEngineering");
                var rows = new List<UpdateEntry>();
                foreach (ManagementObject o in q.Get())
                using (o)
                {
                    DateTime.TryParse(o["InstalledOn"]?.ToString(), out var d);
                    var id = o["HotFixID"]?.ToString() ?? "";
                    rows.Add(new UpdateEntry(d, $"{id} {o["Description"]}".Trim(), "Installed"));
                }
                info.Recent = rows.OrderByDescending(r => r.Date).Take(30).ToList();
                info.LastInstalled = info.Recent.Where(r => r.Date > DateTime.MinValue)
                                                .Select(r => (DateTime?)r.Date).FirstOrDefault();
            }
            catch { }
        }

        return info;
    }
}
```
