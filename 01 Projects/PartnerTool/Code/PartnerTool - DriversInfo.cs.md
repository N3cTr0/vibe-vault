---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DriversInfo.cs
---

# PartnerTool\DriversInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record DriverItem(string Device, string Provider, string Version, string Date, bool? Signed)
{
    public string SignedText => Signed == true ? "Signed" : Signed == false ? "UNSIGNED" : "—";
}

/// <summary>
/// Installed device drivers (Win32_PnPSignedDriver): provider, version, date and signing
/// status — unsigned drivers float to the top as they're the usual troublemakers.
/// </summary>
public static class DriversInfo
{
    public static List<DriverItem> Collect()
    {
        var list = new List<DriverItem>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT DeviceName, DriverProviderName, DriverVersion, DriverDate, IsSigned FROM Win32_PnPSignedDriver");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var dev = o["DeviceName"]?.ToString()?.Trim();
                if (string.IsNullOrEmpty(dev)) continue;
                string date = "";
                try { if (o["DriverDate"] != null)
                        date = ManagementDateTimeConverter.ToDateTime(o["DriverDate"].ToString()).ToString(Dates.Date); }
                catch { }
                bool? signed = o["IsSigned"] is bool b ? b : null;
                list.Add(new DriverItem(dev, o["DriverProviderName"]?.ToString()?.Trim() ?? "—",
                    o["DriverVersion"]?.ToString()?.Trim() ?? "—", date, signed));
            }
        }
        catch { }
        // Unsigned first, then by device name.
        return list
            .GroupBy(d => d.Device + d.Version).Select(g => g.First())   // de-dupe
            .OrderByDescending(d => d.Signed == false)
            .ThenBy(d => d.Device, StringComparer.OrdinalIgnoreCase)
            .ToList();
    }
}
```
