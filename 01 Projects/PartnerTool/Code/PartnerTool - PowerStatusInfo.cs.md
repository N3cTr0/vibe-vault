---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PowerStatusInfo.cs
---

# PartnerTool\PowerStatusInfo.cs

```csharp
using System.Globalization;
using System.Runtime.InteropServices;
using System.Text.RegularExpressions;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>
/// Power-related facts shown under the System Info ▸ Power buttons: where the power is coming
/// from, the active power mode, Fast Startup / Hibernation state, the sleep &amp; display
/// timeouts, and what last woke the machine. All cheap reads (Win32 + registry + a couple of
/// bounded <c>powercfg</c> calls), each guarded so a failure just shows "—".
/// </summary>
public class PowerStatusInfo
{
    public string PowerSource { get; set; } = "—";   // "On AC power" / "On battery (72%)"
    public string PowerMode   { get; set; } = "—";   // Balanced / Best performance …
    public string FastStartup { get; set; } = "—";   // On / Off
    public string Hibernation { get; set; } = "—";   // Enabled / Disabled
    public string Sleep       { get; set; } = "—";   // "30 min (AC) · 15 min (battery)" / Never
    public string DisplayOff  { get; set; } = "—";
    public string LastWake    { get; set; } = "—";

    public static async Task<PowerStatusInfo> CollectAsync()
    {
        var p = new PowerStatusInfo();

        try { p.PowerSource = ReadPowerSource(); } catch { }
        try { p.PowerMode   = global::PartnerTool.PowerMode.ActiveName(); } catch { }
        try { p.FastStartup = ReadDword(@"SYSTEM\CurrentControlSet\Control\Session Manager\Power", "HiberbootEnabled", "On", "Off"); } catch { }

        // powercfg calls in parallel so they cost one round-trip, not several.
        var sleep = ReadTimeoutAsync("SUB_SLEEP", "STANDBYIDLE");
        var disp  = ReadTimeoutAsync("SUB_VIDEO", "VIDEOIDLE");
        var wake  = ReadLastWakeAsync();
        var hib   = ReadHibernationAsync();
        try { await Task.WhenAll(sleep, disp, wake, hib); } catch { }
        try { p.Sleep       = await sleep; } catch { }
        try { p.DisplayOff  = await disp;  } catch { }
        try { p.LastWake    = await wake;  } catch { }
        try { p.Hibernation = await hib;   } catch { }

        return p;
    }

    // ── Power source (Win32) ──────────────────────────────────
    [StructLayout(LayoutKind.Sequential)]
    private struct SYSTEM_POWER_STATUS
    {
        public byte ACLineStatus;       // 0 = battery, 1 = AC, 255 = unknown
        public byte BatteryFlag;        // 128 = no system battery
        public byte BatteryLifePercent; // 0-100, 255 = unknown
        public byte SystemStatusFlag;
        public uint BatteryLifeTime;
        public uint BatteryFullLifeTime;
    }

    [DllImport("kernel32.dll")]
    private static extern bool GetSystemPowerStatus(out SYSTEM_POWER_STATUS s);

    private static string ReadPowerSource()
    {
        if (!GetSystemPowerStatus(out var s)) return "—";
        if ((s.BatteryFlag & 128) != 0) return "On AC power";   // desktop / no battery

        string pct = s.BatteryLifePercent <= 100 ? $" ({s.BatteryLifePercent}%)" : "";
        return s.ACLineStatus switch
        {
            1 => $"On AC power{pct}",
            0 => $"On battery{pct}",
            _ => "—",
        };
    }

    // ── Registry DWORD → on/off text ──────────────────────────
    private static string ReadDword(string subkey, string name, string ifOne, string ifZero)
    {
        using var k = Registry.LocalMachine.OpenSubKey(subkey);
        return k?.GetValue(name) is int i ? (i == 1 ? ifOne : ifZero) : "—";
    }

    // ── Sleep / display timeout via powercfg /q ───────────────
    private static async Task<string> ReadTimeoutAsync(string sub, string setting)
    {
        var outp = await ProcessRunner.RunCaptureAsync("powercfg", $"/q SCHEME_CURRENT {sub} {setting}", 8000);
        int? ac = ParseIndex(outp, "AC");
        int? dc = ParseIndex(outp, "DC");
        if (ac is null && dc is null) return "—";

        static string Fmt(int? secs) =>
            secs is null ? "—" : secs == 0 ? "Never" : secs < 60 ? $"{secs}s" : $"{secs / 60} min";

        // If there's no battery, powercfg still reports a DC value but it's irrelevant — show AC only.
        return ac == dc || dc is null ? Fmt(ac) : $"{Fmt(ac)} (AC) · {Fmt(dc)} (battery)";
    }

    private static int? ParseIndex(string outp, string which)
    {
        var m = Regex.Match(outp, $@"Current {which} Power Setting Index:\s*0x([0-9a-fA-F]+)");
        return m.Success && int.TryParse(m.Groups[1].Value, NumberStyles.HexNumber, CultureInfo.InvariantCulture, out var secs)
            ? secs : null;
    }

    // ── Last wake source via powercfg /lastwake ───────────────
    private static async Task<string> ReadLastWakeAsync()
    {
        var outp = await ProcessRunner.RunCaptureAsync("powercfg", "/lastwake", 8000);
        if (Regex.IsMatch(outp, @"Wake Source Count\s*-\s*0")) return "Clean boot (no wake from sleep)";

        var type = Regex.Match(outp, @"Type:\s*(.+)").Groups[1].Value.Trim();
        if (type.Length == 0) return "—";
        var detail = Regex.Match(outp, @"(?:Friendly Name|Owner|Instance Path):\s*(.+)").Groups[1].Value.Trim();
        return detail.Length == 0 ? type : $"{type} — {detail}";
    }

    // ── Hibernation availability via powercfg /a ──────────────
    private static async Task<string> ReadHibernationAsync()
    {
        var outp = await ProcessRunner.RunCaptureAsync("powercfg", "/a", 8000);
        if (string.IsNullOrWhiteSpace(outp)) return "—";
        // Everything before the "not available" header is the list of available sleep states.
        int idx = outp.IndexOf("not available", StringComparison.OrdinalIgnoreCase);
        var available = idx > 0 ? outp[..idx] : outp;
        return available.Contains("Hibernate", StringComparison.OrdinalIgnoreCase) ? "Enabled" : "Disabled";
    }
}
```
