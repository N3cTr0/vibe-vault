---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PrintersInfo.cs
---

# PartnerTool\PrintersInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record PrinterInfo(string Name, bool Default, bool Offline, string Status, string Port, string Driver)
{
    public string Display => (Default ? "★ " : "") + Name;
    public string Detail  => $"{(Offline ? "Offline" : Status)}   ·   {Port}   ·   {Driver}";
}

/// <summary>Installed printers, the default, and whether each is offline (Win32_Printer).</summary>
public static class PrintersInfo
{
    public static List<PrinterInfo> Collect()
    {
        var list = new List<PrinterInfo>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, Default, WorkOffline, PrinterStatus, PortName, DriverName FROM Win32_Printer");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                string status = Convert.ToInt32(o["PrinterStatus"] ?? 0) switch
                {
                    1 => "Other", 2 => "Unknown", 3 => "Idle", 4 => "Printing",
                    5 => "Warming up", 6 => "Stopped", 7 => "Offline", _ => "—",
                };
                list.Add(new PrinterInfo(
                    o["Name"]?.ToString() ?? "Printer",
                    o["Default"] is bool d && d,
                    o["WorkOffline"] is bool w && w,
                    status,
                    o["PortName"]?.ToString() ?? "",
                    o["DriverName"]?.ToString() ?? ""));
            }
        }
        catch { }
        return list.OrderByDescending(p => p.Default).ThenBy(p => p.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }
}
```
