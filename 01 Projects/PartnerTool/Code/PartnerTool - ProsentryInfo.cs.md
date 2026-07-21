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
    // Atakama routes DNS through a local resolver — when it's active the adapter's DNS servers are
    // set to these loopback addresses.
    private static readonly string[] AtakamaDns = { "127.97.116.97", "127.97.116.98" };

    public static ProsentryReport Collect() => new(
        new List<ManagedTool> { CheckAtakama(), CheckHuntress(), CheckDuo(), CheckAutoElevate() },
        CheckIntune());

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
                        return new("Atakama", true, $"Active — DNS routed via Atakama ({dns})");
            }
        }
        catch { }
        return new("Atakama", false, "Not active (DNS not pointed at Atakama)");
    }

    private static ManagedTool CheckHuntress()
    {
        // Huntress installs the HuntressAgent service (+ HuntressUpdater, and HuntressRio for EDR).
        var (exists, running) = ServiceState("HuntressAgent");
        if (!exists) (exists, running) = ServiceState("HuntressRio");
        if (exists)
            return new("Huntress EDR", running, running ? "Agent running" : "Installed — agent not running");
        if (InstalledNameContains("Huntress"))
            return new("Huntress EDR", true, "Installed");
        return new("Huntress EDR", false, "Not installed");
    }

    private static ManagedTool CheckDuo()
    {
        // Duo Authentication for Windows Logon is a credential provider (no long-running service).
        // Check its DuoCredProv credential-provider subkey or uninstall entry — NOT the bare
        // "SOFTWARE\Duo Security" parent, which can linger as an empty key and false-positive.
        if (RegistryKeyExists(Registry.LocalMachine, @"SOFTWARE\Duo Security\DuoCredProv") ||
            InstalledNameContains("Duo Authentication"))
            return new("Duo", true, "Installed");
        return new("Duo", false, "Not installed");
    }

    private static ManagedTool CheckAutoElevate()
    {
        var (exists, running) = ServiceState("AutoElevateAgent");
        if (exists)
            return new("AutoElevate", running, running ? "Agent running" : "Installed — agent not running");
        if (InstalledNameContains("AutoElevate"))
            return new("AutoElevate", true, "Installed");
        return new("AutoElevate", false, "Not installed");
    }

    // ── Device management (not ProSentry) ─────────────────────────────────

    private static ManagedTool CheckIntune()
    {
        // An MDM enrollment whose ProviderID is "MS DM Server" (and a non-zero EnrollmentType) is a
        // real device enrollment — Intune uses this. The Intune Management Extension service is a
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
                            string.IsNullOrWhiteSpace(upn) ? "Enrolled" : $"Enrolled — {upn}");
                    }
                }
        }
        catch { }
        if (ServiceState("IntuneManagementExtension").exists)
            return new("Intune (MDM)", true, "Management Extension present");
        return new("Intune (MDM)", false, "Not enrolled");
    }

    // ── helpers ───────────────────────────────────────────────────────────

    /// <summary>(exists, running) for a service — names are hardcoded constants, so the WMI path is safe.</summary>
    private static (bool exists, bool running) ServiceState(string name)
    {
        try
        {
            using var svc = new ManagementObject($"Win32_Service.Name='{name}'");
            svc.Get();
            return (true, (svc["State"] as string) == "Running");
        }
        catch { return (false, false); }
    }

    private static bool RegistryKeyExists(RegistryKey root, string path)
    {
        using var k = root.OpenSubKey(path);
        return k != null;
    }

    private static bool InstalledNameContains(string needle)
    {
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
                    if (s?.GetValue("DisplayName") as string is { } dn &&
                        dn.Contains(needle, StringComparison.OrdinalIgnoreCase))
                        return true;
                }
            }
            catch { }
        }
        return false;
    }
}
```
