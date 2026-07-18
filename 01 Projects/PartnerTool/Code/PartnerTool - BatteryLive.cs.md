---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BatteryLive.cs
---

# PartnerTool\BatteryLive.cs

```csharp
using System.Management;

namespace PartnerTool;

/// <summary>A live battery reading: charge rate and estimated runtime, which the static
/// snapshot (charge %, wear, capacities) can't show changing.</summary>
public record BatteryReading(bool Present, bool Charging, bool OnAc, double? RateW, int? RuntimeMin, double? VoltageV)
{
    /// <summary>e.g. "Charging · 31.2 W", "On battery · 12.5 W · ~2h 14m left", "Plugged in".</summary>
    public string Summary
    {
        get
        {
            if (!Present) return "";
            string head = Charging ? "Charging" : OnAc ? "Plugged in (not charging)" : "On battery";
            var parts = new List<string> { head };
            if (RateW is { } w && w > 0) parts.Add($"{w:F1} W");
            if (!OnAc && RuntimeMin is { } m && m > 0)
                parts.Add($"~{m / 60}h {m % 60:00}m left");
            return string.Join("  ·  ", parts);
        }
    }
}

/// <summary>
/// Live battery telemetry from WMI (no kernel driver): charge/discharge rate from
/// <c>root\wmi BatteryStatus</c> and estimated runtime from <c>Win32_Battery</c>. Cheap enough
/// to poll on the live timer; returns <c>Present=false</c> on desktops.
/// </summary>
public static class BatteryLive
{
    public static BatteryReading Sample()
    {
        bool present = false; bool charging = false, onAc = true; int? runtime = null;
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT BatteryStatus, EstimatedRunTime FROM Win32_Battery");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                present = true;
                int status = o["BatteryStatus"] is { } s ? Convert.ToInt32(s) : 0;
                onAc = status != 1;                       // 1 = discharging
                charging = status is 2 or 6 or 7 or 8 or 9;
                // EstimatedRunTime is in minutes; 71582788 is the "unknown" sentinel.
                if (!onAc && o["EstimatedRunTime"] is { } rt)
                {
                    int m = Convert.ToInt32(rt);
                    if (m is > 0 and < 71582788) runtime = m;
                }
                break;
            }
        }
        catch { }

        if (!present) return new BatteryReading(false, false, true, null, null, null);

        double? rateW = null, voltage = null;
        try
        {
            using var q = new ManagementObjectSearcher(@"root\wmi",
                "SELECT Charging, Discharging, PowerOnline, ChargeRate, DischargeRate, Voltage FROM BatteryStatus");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                bool ch  = o["Charging"]    as bool? ?? false;
                bool dis = o["Discharging"] as bool? ?? false;
                if (o["PowerOnline"] is bool po) onAc = po;
                charging = ch;
                double charge = o["ChargeRate"]    is { } cr ? Convert.ToDouble(cr) : 0;
                double dischg = o["DischargeRate"] is { } dr ? Convert.ToDouble(dr) : 0;
                if (ch && charge > 0)        rateW = Math.Round(charge / 1000.0, 1);   // mW → W
                else if (dis && dischg > 0)  rateW = Math.Round(dischg / 1000.0, 1);
                if (o["Voltage"] is { } vv) { double mv = Convert.ToDouble(vv); if (mv > 0) voltage = Math.Round(mv / 1000.0, 1); }  // mV → V
                break;
            }
        }
        catch { }

        return new BatteryReading(true, charging, onAc, rateW, runtime, voltage);
    }
}
```
