---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\WifiInfo.cs
---

# PartnerTool\WifiInfo.cs

```csharp
using System.Net.NetworkInformation;
using System.Text.RegularExpressions;

namespace PartnerTool;

/// <summary>
/// Wi-Fi connection state and saved network profiles. State comes from <see cref="NetworkInterface"/>
/// (no Location permission needed); the SSID/signal are intentionally not read — Windows 11 gates
/// those behind Location and they aren't worth the friction. Saved profiles + the (sensitive)
/// password reveal use netsh and don't need Location.
/// </summary>
public static class WifiInfo
{
    /// <summary>True when a wireless adapter is connected (a Wireless80211 interface that's Up).</summary>
    public static bool IsConnected()
    {
        try
        {
            return NetworkInterface.GetAllNetworkInterfaces().Any(n =>
                n.NetworkInterfaceType == NetworkInterfaceType.Wireless80211 &&
                n.OperationalStatus == OperationalStatus.Up);
        }
        catch { return false; }
    }

    public static async Task<List<string>> GetProfilesAsync()
    {
        var text = await ProcessRunner.RunCaptureAsync("netsh.exe", "wlan show profiles");
        var names = new List<string>();
        foreach (Match m in Regex.Matches(text, @"All User Profile\s*:\s*(.+)", RegexOptions.IgnoreCase))
            names.Add(m.Groups[1].Value.Trim());
        foreach (Match m in Regex.Matches(text, @"User Profile\s*:\s*(.+)", RegexOptions.IgnoreCase))
        {
            var n = m.Groups[1].Value.Trim();
            if (!names.Contains(n)) names.Add(n);
        }
        return names.Distinct().OrderBy(n => n, StringComparer.OrdinalIgnoreCase).ToList();
    }

    /// <summary>Reveal a saved network's password (key=clear). Sensitive — caller must gate this.</summary>
    public static async Task<string> GetPasswordAsync(string profile)
    {
        // Strip quotes so a hostile SSID can't break out of the quoted netsh argument.
        profile = profile.Replace("\"", "");
        var text = await ProcessRunner.RunCaptureAsync("netsh.exe", $"wlan show profile name=\"{profile}\" key=clear");
        var m = Regex.Match(text, @"Key Content\s*:\s*(.+)", RegexOptions.IgnoreCase);
        return m.Success ? m.Groups[1].Value.Trim() : "(no saved password — open network or stored elsewhere)";
    }
}
```
