---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DevModeInfo.cs
---

# PartnerTool\DevModeInfo.cs

```csharp
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>
/// Windows Developer Mode - the Settings ▸ System ▸ For developers toggle. Settings only offers it
/// to an administrator, so on a machine where the signed-in user is standard a tech otherwise has to
/// sign in as an admin just to flip it. It is two HKLM values, and the tool already runs elevated.
/// </summary>
public static class DevModeInfo
{
    private const string Key         = @"SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock";
    private const string PolicyKey   = @"SOFTWARE\Policies\Microsoft\Windows\Appx";
    private const string DevLicense  = "AllowDevelopmentWithoutDevLicense";
    private const string TrustedApps = "AllowAllTrustedApps";

    public static bool IsEnabled()
    {
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(Key);
            return k?.GetValue(DevLicense) is int v && v == 1;
        }
        catch { return false; }
    }

    /// <summary>
    /// The "Allow development of Windows Store apps" GPO writes the same two values under Policies,
    /// which win over the Settings ones. Worth saying so - otherwise a policy-managed machine looks
    /// like the toggle silently refused to stick.
    /// </summary>
    private static int? PolicyValue()
    {
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(PolicyKey);
            return k?.GetValue(DevLicense) as int?;
        }
        catch { return null; }
    }

    public static (bool ok, string message) Set(bool on)
    {
        try
        {
            using var k = Registry.LocalMachine.CreateSubKey(Key, writable: true);
            if (k == null) return (false, "● Couldn't open AppModelUnlock.");
            k.SetValue(DevLicense,  on ? 1 : 0, RegistryValueKind.DWord);
            k.SetValue(TrustedApps, on ? 1 : 0, RegistryValueKind.DWord);
        }
        catch (Exception ex) { return (false, "● " + ex.Message); }

        if (PolicyValue() is { } p && (p == 1) != on)
            return (true, $"● Set, but group policy forces Developer Mode {(p == 1 ? "On" : "Off")} - " +
                          "the policy wins until it's changed on the DC.");

        return (true, on
            ? "● Developer Mode is On - apps can be installed from any source, including loose files."
            : "● Developer Mode is Off.");
    }
}
```
