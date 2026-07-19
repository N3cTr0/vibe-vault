---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SoftwareInventory.cs
---

# PartnerTool\SoftwareInventory.cs

```csharp
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>One installed program from the uninstall registry (HKLM 64/32-bit + HKCU).</summary>
public record InstalledApp(string Name, string Version, string Publisher, string InstallDate, string Uninstall, bool Machine);

/// <summary>
/// The installed-software inventory, read from the Add/Remove Programs registry keys. Shared by
/// the Software page (install/uninstall) and the diagnostics report so both see the same list.
/// </summary>
public static class SoftwareInventory
{
    public static List<InstalledApp> Collect()
    {
        var seen = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        var apps = new List<InstalledApp>();

        var paths = new[]
        {
            (RegistryHive.LocalMachine, RegistryView.Registry64, @"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"),
            (RegistryHive.LocalMachine, RegistryView.Registry32, @"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"),
            (RegistryHive.CurrentUser,  RegistryView.Registry64, @"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"),
        };

        foreach (var (hive, view, path) in paths)
        {
            try
            {
                using var root = RegistryKey.OpenBaseKey(hive, view);
                using var key  = root.OpenSubKey(path);
                if (key == null) continue;

                foreach (var name in key.GetSubKeyNames())
                {
                    using var sub = key.OpenSubKey(name);
                    if (sub == null) continue;

                    var display = sub.GetValue("DisplayName")?.ToString();
                    if (string.IsNullOrWhiteSpace(display)) continue;
                    if (sub.GetValue("SystemComponent") is int sc && sc == 1) continue;
                    if (sub.GetValue("ParentKeyName") != null) continue;
                    if (!seen.Add(display)) continue;

                    apps.Add(new InstalledApp(
                        display,
                        sub.GetValue("DisplayVersion")?.ToString() ?? "",
                        sub.GetValue("Publisher")?.ToString() ?? "",
                        ParseDate(sub.GetValue("InstallDate")?.ToString()),
                        (sub.GetValue("QuietUninstallString") ?? sub.GetValue("UninstallString"))?.ToString() ?? "",
                        // HKLM uninstall strings are admin-only-writable, so safe to run elevated.
                        hive == RegistryHive.LocalMachine));
                }
            }
            catch { }
        }

        return apps.OrderBy(a => a.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    public static string ParseDate(string? raw)
    {
        if (raw?.Length == 8 &&
            DateTime.TryParseExact(raw, "yyyyMMdd", null,
                System.Globalization.DateTimeStyles.None, out var dt))
            return dt.ToString(Dates.Date);
        return "";
    }
}
```
