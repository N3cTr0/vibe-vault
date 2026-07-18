---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SystemInfo.cs
---

# PartnerTool\SystemInfo.cs

```csharp
using Microsoft.Win32;
using System.Management;
using System.Net.NetworkInformation;
using System.Runtime.InteropServices;

namespace PartnerTool;

public class SystemInfo
{
    public string Hostname           { get; init; } = string.Empty;
    public string LoggedInUser       { get; init; } = string.Empty;
    public string Domain             { get; init; } = string.Empty;
    public string Timezone           { get; init; } = string.Empty;
    public string ManufacturerModel  { get; init; } = string.Empty;
    public string SerialNumber       { get; init; } = string.Empty;
    public string OsVersion          { get; init; } = string.Empty;
    public string WindowsVersion     { get; init; } = string.Empty;
    public string KernelVersion      { get; init; } = string.Empty;
    public string KernelArchitecture { get; init; } = string.Empty;

    public static string GetManufacturer()
    {
        try
        {
            using var cs = new ManagementObjectSearcher("SELECT Manufacturer FROM Win32_ComputerSystem");
            foreach (ManagementObject obj in cs.Get())
                return obj["Manufacturer"]?.ToString()?.Trim() ?? string.Empty;
        }
        catch { }
        return string.Empty;
    }

    /// <summary>Manufacturer + model in one WMI read — needed to tell a Surface from a Microsoft VM
    /// (and any OEM box from a virtual machine) when picking the vendor update tool.</summary>
    public static (string Manufacturer, string Model) GetMakeModel()
    {
        try
        {
            using var cs = new ManagementObjectSearcher("SELECT Manufacturer, Model FROM Win32_ComputerSystem");
            foreach (ManagementObject obj in cs.Get())
                return (obj["Manufacturer"]?.ToString()?.Trim() ?? string.Empty,
                        obj["Model"]?.ToString()?.Trim() ?? string.Empty);
        }
        catch { }
        return (string.Empty, string.Empty);
    }

    public static SystemInfo Collect()
    {
        string hostname     = Environment.MachineName;
        string user         = $@"{Environment.UserDomainName}\{Environment.UserName}";
        string timezone     = TimeZoneInfo.Local.DisplayName;
        string kernelArch   = RuntimeInformation.OSArchitecture.ToString();
        string kernelVer    = Environment.OSVersion.VersionString;
        string osVersion    = RuntimeInformation.OSDescription;
        string domain       = string.Empty;
        string manufacturer = string.Empty;
        string model        = string.Empty;
        string serial       = string.Empty;
        string winVer       = string.Empty;

        try { domain = IPGlobalProperties.GetIPGlobalProperties().DomainName; } catch { }

        try
        {
            using var cs = new ManagementObjectSearcher("SELECT Caption FROM Win32_OperatingSystem");
            foreach (ManagementObject obj in cs.Get())
            {
                var caption = obj["Caption"]?.ToString()?.Trim();
                if (!string.IsNullOrEmpty(caption)) osVersion = caption;
            }
        }
        catch { }

        // Feature-update version (e.g. 22H2 / 23H2 / 24H2) — what winver shows.
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows NT\CurrentVersion");
            winVer = k?.GetValue("DisplayVersion") as string
                  ?? k?.GetValue("ReleaseId") as string
                  ?? string.Empty;
        }
        catch { }

        try
        {
            using var cs = new ManagementObjectSearcher("SELECT Manufacturer, Model, Domain FROM Win32_ComputerSystem");
            foreach (ManagementObject obj in cs.Get())
            {
                manufacturer = obj["Manufacturer"]?.ToString()?.Trim() ?? string.Empty;
                model        = obj["Model"]?.ToString()?.Trim() ?? string.Empty;
                var wmiDomain = obj["Domain"]?.ToString()?.Trim();
                if (!string.IsNullOrEmpty(wmiDomain)) domain = wmiDomain;
            }
        }
        catch { }

        try
        {
            using var bios = new ManagementObjectSearcher("SELECT SerialNumber FROM Win32_BIOS");
            foreach (ManagementObject obj in bios.Get())
                serial = obj["SerialNumber"]?.ToString()?.Trim() ?? string.Empty;
        }
        catch { }

        return new SystemInfo
        {
            Hostname           = hostname,
            LoggedInUser       = user,
            Domain             = string.IsNullOrWhiteSpace(domain) ? "Not domain joined" : domain,
            Timezone           = timezone,
            ManufacturerModel  = string.IsNullOrWhiteSpace(manufacturer) ? "Unknown" : $"{manufacturer}, {model}",
            SerialNumber       = string.IsNullOrWhiteSpace(serial) ? "Unknown" : serial,
            OsVersion          = osVersion,
            WindowsVersion     = string.IsNullOrWhiteSpace(winVer) ? "Unknown" : winVer,
            KernelVersion      = kernelVer,
            KernelArchitecture = kernelArch,
        };
    }
}
```
