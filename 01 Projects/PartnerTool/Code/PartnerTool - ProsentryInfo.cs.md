---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ProsentryInfo.cs
---

# PartnerTool\ProsentryInfo.cs

```csharp
using System.Management;
using System.Net.NetworkInformation;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>One managed-security/agent tool and whether it's active on this PC.</summary>
public record ManagedTool(string Name, bool Active, string Detail);

/// <summary>ProSentry stack (the four PCI agents) + device-management (Intune) status.</summary>
public record ProsentryReport(List<ManagedTool> Tools, ManagedTool Intune);

/// <summary>
/// Detects the PCI ProSentry security stack and Intune enrollment so a tech can see at a glance
/// what's protecting/managing a machine. All checks are read-only registry / WMI / network reads.
/// </summary>
public static class ProsentryInfo
{
    // Atakama routes DNS through a local resolver - when it's active the adapter's DNS servers are
    // set to these loopback addresses.
    private static readonly string[] AtakamaDns = { "127.97.116.97", "127.97.116.98" };

    /// <summary>
    /// One Win32_Service enumeration and at most one walk of the uninstall hives feed every check
    /// below. Asking WMI for each service by name cost a separate round trip per agent, and each
    /// InstalledNameContains re-opened every uninstall subkey on the machine.
    /// </summary>
    public static ProsentryReport Collect()
    {
        var services  = LoadServices();
        var installed = new Lazy<List<string>>(LoadInstalledNames);
        return new(
            new List<ManagedTool>
            {
                CheckAtakama(),
                CheckHuntress(services, installed),
                CheckDuo(installed),
                CheckAutoElevate(services, installed),
            },
            CheckIntune(services));
    }

    // ── ProSentry agents ──────────────────────────────────────────────────

    private static ManagedTool CheckAtakama()
    {
        try
        {
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
            {
                if (ni.OperationalStatus != OperationalStatus.Up) continue;
                foreach (var dns in ni.GetIPProperties().DnsAddresses)
                    if (AtakamaDns.Contains(dns.ToString()))
                        return new("Atakama", true, $"Active - DNS routed via Atakama ({dns})");
            }
        }
        catch { }
        return new("Atakama", false, "Not active (DNS not pointed at Atakama)");
    }

    private static ManagedTool CheckHuntress(Dictionary<string, bool> services, Lazy<List<string>> installed)
    {
        // Huntress installs the HuntressAgent service (+ HuntressUpdater, and HuntressRio for EDR).
        if (!services.TryGetValue("HuntressAgent", out var running) &&
            !services.TryGetValue("HuntressRio",   out running))
        {
            if (InstalledNameContains(installed, "Huntress"))
                return new("Huntress EDR", true, "Installed");
            return new("Huntress EDR", false, "Not installed");
        }
        return new("Huntress EDR", running, running ? "Agent running" : "Installed - agent not running");
    }

    private static ManagedTool CheckDuo(Lazy<List<string>> installed)
    {
        // Duo Authentication for Windows Logon is a credential provider (no long-running service).
        // Check its DuoCredProv credential-provider subkey or uninstall entry - NOT the bare
        // "SOFTWARE\Duo Security" parent, which can linger as an empty key and false-positive.
        if (RegistryKeyExists(Registry.LocalMachine, @"SOFTWARE\Duo Security\DuoCredProv") ||
            InstalledNameContains(installed, "Duo Authentication"))
            return new("Duo", true, "Installed");
        return new("Duo", false, "Not installed");
    }

    private static ManagedTool CheckAutoElevate(Dictionary<string, bool> services, Lazy<List<string>> installed)
    {
        if (services.TryGetValue("AutoElevateAgent", out var running))
            return new("AutoElevate", running, running ? "Agent running" : "Installed - agent not running");
        if (InstalledNameContains(installed, "AutoElevate"))
            return new("AutoElevate", true, "Installed");
        return new("AutoElevate", false, "Not installed");
    }

    // ── Device management (not ProSentry) ─────────────────────────────────

    private static ManagedTool CheckIntune(Dictionary<string, bool> services)
    {
        // An MDM enrollment whose ProviderID is "MS DM Server" (and a non-zero EnrollmentType) is a
        // real device enrollment - Intune uses this. The Intune Management Extension service is a
        // secondary signal (present on Intune-managed devices that got Win32/PowerShell policies).
        try
        {
            using var root = Registry.LocalMachine.OpenSubKey(@"SOFTWARE\Microsoft\Enrollments");
            if (root != null)
                foreach (var sub in root.GetSubKeyNames())
                {
                    using var k = root.OpenSubKey(sub);
                    if (k?.GetValue("ProviderID") as string is { } prov &&
                        prov.Equals("MS DM Server", StringComparison.OrdinalIgnoreCase) &&
                        k.GetValue("EnrollmentType") is int et && et != 0)
                    {
                        var upn = k.GetValue("UPN") as string;
                        return new("Intune (MDM)", true,
                            string.IsNullOrWhiteSpace(upn) ? "Enrolled" : $"Enrolled - {upn}");
                    }
                }
        }
        catch { }
        if (services.ContainsKey("IntuneManagementExtension"))
            return new("Intune (MDM)", true, "Management Extension present");
        return new("Intune (MDM)", false, "Not enrolled");
    }

    // ── helpers ───────────────────────────────────────────────────────────

    /// <summary>Every installed service, name -> is it running. One enumeration for the whole report.</summary>
    private static Dictionary<string, bool> LoadServices()
    {
        var map = new Dictionary<string, bool>(StringComparer.OrdinalIgnoreCase);
        try
        {
            using var q = new ManagementObjectSearcher("SELECT Name, State FROM Win32_Service");
            foreach (ManagementObject o in q.Get())
                using (o)
                {
                    if (o["Name"]?.ToString() is { Length: > 0 } n)
                        map[n] = (o["State"] as string) == "Running";
                }
        }
        catch { }
        return map;
    }

    private static bool RegistryKeyExists(RegistryKey root, string path)
    {
        using var k = root.OpenSubKey(path);
        return k != null;
    }

    /// <summary>Every uninstall-entry DisplayName, read once (both registry views).</summary>
    private static List<string> LoadInstalledNames()
    {
        var names = new List<string>();
        string[] paths =
        {
            @"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
            @"SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
        };
        foreach (var path in paths)
        {
            try
            {
                using var k = Registry.LocalMachine.OpenSubKey(path);
                if (k == null) continue;
                foreach (var subName in k.GetSubKeyNames())
                {
                    using var s = k.OpenSubKey(subName);
                    if (s?.GetValue("DisplayName") as string is { } dn) names.Add(dn);
                }
            }
            catch { }
        }
        return names;
    }

    private static bool InstalledNameContains(Lazy<List<string>> installed, string needle)
        => installed.Value.Any(dn => dn.Contains(needle, StringComparison.OrdinalIgnoreCase));
}
```
