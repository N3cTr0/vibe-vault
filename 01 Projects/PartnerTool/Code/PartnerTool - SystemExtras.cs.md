---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SystemExtras.cs
---

# PartnerTool\SystemExtras.cs

```csharp
using System.Management;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>
/// Odds-and-ends a tech occasionally needs: the active power mode, page-file configuration
/// and the system proxy. All cheap WMI/registry reads.
/// </summary>
public class SystemExtras
{
    public string PageFile  { get; set; } = "Unknown";
    public string Proxy     { get; set; } = "Not configured";
    public string PowerPlan { get; set; } = "Unknown";

    public static SystemExtras Collect()
    {
        var x = new SystemExtras();

        // ── Active Windows "Power Mode" (the Settings slider) ──
        x.PowerPlan = PowerMode.ActiveName();

        // ── Page file ─────────────────────────────────────────
        try
        {
            bool managed = false;
            using (var cs = new ManagementObjectSearcher("SELECT AutomaticManagedPagefile FROM Win32_ComputerSystem"))
                foreach (ManagementObject o in cs.Get()) { managed = o["AutomaticManagedPagefile"] is bool b && b; break; }

            string detail = "";
            using (var pf = new ManagementObjectSearcher(
                "SELECT Name, AllocatedBaseSize, CurrentUsage, PeakUsage FROM Win32_PageFileUsage"))
                foreach (ManagementObject o in pf.Get())
                {
                    detail = $"{o["Name"]} — {o["AllocatedBaseSize"]} MB allocated, " +
                             $"{o["CurrentUsage"]} MB in use (peak {o["PeakUsage"]} MB)";
                    break;
                }

            x.PageFile = (managed ? "System managed. " : "") + (detail.Length > 0 ? detail : "No page file");
        }
        catch { }

        // ── Proxy (per-user WinINET) ──────────────────────────
        try
        {
            using var k = Registry.CurrentUser.OpenSubKey(
                @"Software\Microsoft\Windows\CurrentVersion\Internet Settings");
            bool on = k?.GetValue("ProxyEnable") is int v && v == 1;
            var server = k?.GetValue("ProxyServer") as string;
            var auto   = k?.GetValue("AutoConfigURL") as string;
            if (on && !string.IsNullOrWhiteSpace(server)) x.Proxy = $"Manual proxy: {server}";
            else if (!string.IsNullOrWhiteSpace(auto))    x.Proxy = $"Auto-config script: {auto}";
            else                                          x.Proxy = "Direct (no proxy)";
        }
        catch { }

        return x;
    }
}
```
