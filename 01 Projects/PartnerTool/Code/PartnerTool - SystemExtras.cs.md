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
    /// <summary>Flat one-line form - what the text/HTML reports print.</summary>
    public string PageFile  { get; set; } = "Unknown";
    /// <summary>"System managed" / "Manually configured" / "No page file" - line 1 of the UI form.</summary>
    public string PageFileMode  { get; set; } = "";
    /// <summary>Page-file path, e.g. C:\pagefile.sys - line 2 of the UI form.</summary>
    public string PageFilePath  { get; set; } = "";
    /// <summary>Sizes, e.g. "4864 MB allocated, 30 MB in use (peak 38 MB)" - line 3 of the UI form.</summary>
    public string PageFileUsage { get; set; } = "";
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

            // Kept as three parts so the UI can stack them on separate lines (the single
            // concatenated string overflowed the Performance card), while the reports below
            // still get the flat one-liner they have always printed.
            string path = "", usage = "";
            using (var pf = new ManagementObjectSearcher(
                "SELECT Name, AllocatedBaseSize, CurrentUsage, PeakUsage FROM Win32_PageFileUsage"))
                foreach (ManagementObject o in pf.Get())
                {
                    path  = $"{o["Name"]}";
                    usage = $"{o["AllocatedBaseSize"]} MB allocated, " +
                            $"{o["CurrentUsage"]} MB in use (peak {o["PeakUsage"]} MB)";
                    break;
                }

            x.PageFileMode  = path.Length == 0 ? "No page file"
                            : managed          ? "System managed"
                                               : "Manually configured";
            x.PageFilePath  = path;
            x.PageFileUsage = usage;

            string detail = path.Length > 0 ? $"{path} - {usage}" : "No page file";
            x.PageFile = (managed ? "System managed. " : "") + detail;
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
