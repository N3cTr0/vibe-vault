---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DisplaysInfo.cs
---

# PartnerTool\DisplaysInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record DisplayInfo(string Name, string Manufacturer, string Serial, int? Year, string Size)
{
    // Column values for the System Info monitors table (Make / Model / Size / Year / Serial).
    public string MakeText   => string.IsNullOrWhiteSpace(Manufacturer) ? "—" : Manufacturer;
    public string ModelText  => string.IsNullOrWhiteSpace(Name) ? "—" : Name;
    public string SizeText   => string.IsNullOrWhiteSpace(Size) ? "—" : Size;
    public string YearText   => Year is { } y ? y.ToString() : "—";
    public string SerialText => string.IsNullOrWhiteSpace(Serial) ? "—" : Serial;

    public string Detail
    {
        get
        {
            var parts = new List<string>();
            if (!string.IsNullOrWhiteSpace(Manufacturer)) parts.Add(Manufacturer);
            if (Size != "—" && Size.Length > 0)           parts.Add(Size);
            if (Year is { } y)                            parts.Add(y.ToString());
            if (!string.IsNullOrWhiteSpace(Serial))       parts.Add($"S/N {Serial}");
            return string.Join("   ·   ", parts);
        }
    }
}

/// <summary>
/// Connected monitors, decoded from their EDID (root\wmi WmiMonitorID + basic display
/// params): friendly name, manufacturer, serial, year of manufacture and physical size.
/// </summary>
public static class DisplaysInfo
{
    public static List<DisplayInfo> Collect()
    {
        // Physical size (diagonal inches) keyed by monitor instance name.
        var sizes = new Dictionary<string, string>();
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT InstanceName, MaxHorizontalImageSize, MaxVerticalImageSize FROM WmiMonitorBasicDisplayParams");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                double hCm = Convert.ToDouble(o["MaxHorizontalImageSize"] ?? 0);
                double vCm = Convert.ToDouble(o["MaxVerticalImageSize"] ?? 0);
                if (hCm > 0 && vCm > 0)
                {
                    double inches = Math.Sqrt(hCm * hCm + vCm * vCm) / 2.54;
                    sizes[o["InstanceName"]?.ToString() ?? ""] = $"{inches:F1}\"";
                }
            }
        }
        catch { }

        var list = new List<DisplayInfo>();
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT InstanceName, ManufacturerName, UserFriendlyName, SerialNumberID, YearOfManufacture FROM WmiMonitorID");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                string inst   = o["InstanceName"]?.ToString() ?? "";
                string name   = Decode(o["UserFriendlyName"]);
                string maker  = Decode(o["ManufacturerName"]);
                string serial = Decode(o["SerialNumberID"]);
                int? year = o["YearOfManufacture"] is { } y && Convert.ToInt32(y) > 1990 ? Convert.ToInt32(y) : null;
                list.Add(new DisplayInfo(
                    string.IsNullOrWhiteSpace(name) ? "Display" : name,
                    maker, serial, year,
                    sizes.TryGetValue(inst, out var s) ? s : "—"));
            }
        }
        catch { }
        return list;
    }

    // EDID string fields come back as UInt16[] of character codes, 0-terminated.
    private static string Decode(object? raw)
    {
        if (raw is not ushort[] arr) return "";
        var chars = arr.TakeWhile(c => c != 0).Select(c => (char)c).ToArray();
        return new string(chars).Trim();
    }
}
```
