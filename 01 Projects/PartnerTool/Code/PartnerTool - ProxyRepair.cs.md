---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ProxyRepair.cs
---

# PartnerTool\ProxyRepair.cs

```csharp
using System.Runtime.InteropServices;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>
/// Repairs for the WinINET proxy / auto-detect (WPAD) settings that firewalls like SonicWall like to
/// flip, which breaks Office/Outlook auto-discovery and MAPI-over-HTTP. "Automatically detect
/// settings" lives as bit 0x08 of the flags byte (offset 8) in the binary DefaultConnectionSettings
/// blob — there's no friendlier registry value for it. All of this is per-user (HKCU), so it applies
/// to the signed-in user running the tool.
/// </summary>
public static class ProxyRepair
{
    private const string ConnPath = @"Software\Microsoft\Windows\CurrentVersion\Internet Settings\Connections";
    private const string IsPath   = @"Software\Microsoft\Windows\CurrentVersion\Internet Settings";

    public static bool AutoDetectOn()
    {
        try
        {
            using var k = Registry.CurrentUser.OpenSubKey(ConnPath);
            if (k?.GetValue("DefaultConnectionSettings") is byte[] d && d.Length > 8)
                return (d[8] & 0x08) != 0;
        }
        catch { }
        return false;
    }

    public static string CurrentState()
    {
        string proxy = "none";
        try
        {
            using var k = Registry.CurrentUser.OpenSubKey(IsPath);
            bool on = k?.GetValue("ProxyEnable") is int v && v == 1;
            var server = k?.GetValue("ProxyServer") as string;
            if (on && !string.IsNullOrWhiteSpace(server)) proxy = server;
        }
        catch { }
        return $"Auto-detect (WPAD): {(AutoDetectOn() ? "On" : "Off")}   ·   Manual proxy: {proxy}";
    }

    /// <summary>Toggle "Automatically detect settings" off then back on (the known SonicWall fix).</summary>
    public static (bool ok, string message) ResetAutoDetect()
    {
        try
        {
            SetAutoDetect(false);
            System.Threading.Thread.Sleep(300);
            SetAutoDetect(true);
            return (true, "Done — toggled off then on.  " + CurrentState());
        }
        catch (Exception ex) { return (false, ex.Message); }
    }

    private static void SetAutoDetect(bool on)
    {
        using var key = Registry.CurrentUser.CreateSubKey(ConnPath);
        foreach (var name in new[] { "DefaultConnectionSettings", "SavedLegacySettings" })
        {
            var data = key.GetValue(name) as byte[];
            if (data is null || data.Length < 9)
            {
                data = new byte[60];
                data[0] = 0x46;   // WinINET settings version
                data[4] = 1;      // change counter
                data[8] = 0x09;   // flags: direct (0x01) + auto-detect (0x08)
            }
            data[8] = (byte)(on ? data[8] | 0x08 : data[8] & ~0x08);
            // Bump the change counter (LE uint at offset 4) so WinINET reloads the blob.
            uint counter = BitConverter.ToUInt32(data, 4) + 1;
            BitConverter.GetBytes(counter).CopyTo(data, 4);
            key.SetValue(name, data, RegistryValueKind.Binary);
        }
        NotifyWinInet();
    }

    [DllImport("wininet.dll", SetLastError = true)]
    private static extern bool InternetSetOption(IntPtr hInternet, int dwOption, IntPtr lpBuffer, int dwBufferLength);

    private static void NotifyWinInet()
    {
        try { InternetSetOption(IntPtr.Zero, 39, IntPtr.Zero, 0); } catch { }   // INTERNET_OPTION_SETTINGS_CHANGED
        try { InternetSetOption(IntPtr.Zero, 37, IntPtr.Zero, 0); } catch { }   // INTERNET_OPTION_REFRESH
    }
}
```
