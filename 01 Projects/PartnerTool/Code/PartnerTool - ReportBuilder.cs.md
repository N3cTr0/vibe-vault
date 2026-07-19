---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ReportBuilder.cs
---

# PartnerTool\ReportBuilder.cs

```csharp
using System.Net;
using System.Text;

namespace PartnerTool;

/// <summary>
/// Builds the exported system report — a dark-themed, branded HTML page and a plain-text
/// version — covering everything the tool collects (see <see cref="FullReportData"/>). Handy as
/// an "as-found" record of an old PC when setting up a new one.
///
/// GOING FORWARD: a new collector should get a section here so it lands in the report.
/// </summary>
public static class ReportBuilder
{
    // ──────────────────────────────────────────────────────────────────────
    //  PLAIN TEXT
    // ──────────────────────────────────────────────────────────────────────
    public static string BuildText(FullReportData d)
    {
        var s = d.Snap;
        var info = s.Info; var perf = s.Perf; var sec = s.Security; var hw = s.Hardware;
        var sb = new StringBuilder();
        void H(string t) { sb.AppendLine(); sb.AppendLine(t); sb.AppendLine(new string('-', t.Length)); }

        sb.AppendLine("PARTNER SUPPORT TOOL — SYSTEM REPORT");
        sb.AppendLine($"Generated: {s.CapturedAt:MM/dd/yyyy} at {s.CapturedAt:HH:mm:ss}");
        sb.AppendLine(new string('=', 60));

        H("IDENTITY");
        sb.AppendLine($"  Hostname       : {info.Hostname}");
        sb.AppendLine($"  Logged in user : {info.LoggedInUser}");
        sb.AppendLine($"  Domain         : {info.Domain}");
        sb.AppendLine($"  Timezone       : {info.Timezone}");

        H("DEVICE JOIN");
        sb.AppendLine($"  Azure AD joined  : {YesNo(s.Aad.AzureAdJoined)}");
        sb.AppendLine($"  Domain joined    : {YesNo(s.Aad.DomainJoined)}");
        sb.AppendLine($"  Enterprise joined: {YesNo(s.Aad.EnterpriseJoined)}");
        sb.AppendLine($"  Tenant           : {s.Aad.TenantName}");
        sb.AppendLine($"  Device ID        : {s.Aad.DeviceId}");

        H("HARDWARE");
        sb.AppendLine($"  Manufacturer/Model : {info.ManufacturerModel}");
        sb.AppendLine($"  Serial number      : {info.SerialNumber}");
        sb.AppendLine($"  Processor          : {perf.CpuName} ({perf.Cores}C/{perf.Threads}T, {perf.ClockText})");
        sb.AppendLine($"  Memory             : {hw.TotalMemoryGb:F0} GB ({hw.SlotsUsed}/{hw.SlotsTotal} slots, max {hw.MaxMemoryGb:F0} GB)");
        foreach (var m in hw.Memory)
            sb.AppendLine($"    • {m.Slot}: {m.SizeGb:F0} GB @ {m.SpeedMhz} MHz ({m.Maker})");
        foreach (var disk in hw.Disks)
            sb.AppendLine($"  Disk               : {disk.Model} [{disk.Type}] {disk.SizeGb:F0} GB — {disk.Health}" +
                          (disk.HasSmart ? $" ({disk.SmartText})" : ""));
        foreach (var v in hw.Volumes)
            sb.AppendLine($"  Volume             : {v.Letter} {v.Label} [{v.FileSystem}] {v.UsedGb:F0}/{v.TotalGb:F0} GB used ({v.UsedPct:F0}%)");
        foreach (var g in hw.Gpus)
            sb.AppendLine($"  Graphics           : {g.Name}" + (g.VramGb > 0 ? $", {g.VramGb:F1} GB" : "") +
                          $" — driver {g.Driver} ({g.DriverDate}){(string.IsNullOrEmpty(g.Resolution) ? "" : $", {g.Resolution}")}");
        sb.AppendLine($"  BIOS               : {hw.BiosVersion} ({hw.BiosDate})  ·  {hw.BootMode}  ·  TPM {hw.Tpm}");
        if (hw.BatteryWearPct is { } bw)
            sb.AppendLine($"  Battery            : {bw}% wear (design {hw.BatteryDesignMwh} mWh, full {hw.BatteryFullMwh} mWh)");
        if (s.Temps.CpuTemp is { } ct) sb.AppendLine($"  CPU temperature    : {ct:F0} °C");
        if (s.Temps.GpuTemp is { } gt) sb.AppendLine($"  GPU temperature    : {gt:F0} °C");

        H("OPERATING SYSTEM");
        sb.AppendLine($"  OS version   : {info.OsVersion}");
        sb.AppendLine($"  Feature ver. : {info.WindowsVersion}");
        sb.AppendLine($"  Kernel       : {info.KernelVersion} ({info.KernelArchitecture})");
        sb.AppendLine($"  Last patched : {(s.Updates.LastInstalled is { } lp ? lp.ToString(Dates.Date) : "Unknown")}");

        H("PERFORMANCE");
        sb.AppendLine($"  Memory : {perf.RamUsedGb:F1} / {perf.RamTotalGb:F1} GB ({perf.RamPct:F0}%)");
        sb.AppendLine($"  Disk C:: {perf.DiskUsedGb:F0} / {perf.DiskTotalGb:F0} GB ({perf.DiskPct:F0}%)");
        sb.AppendLine($"  Uptime : {perf.UptimeText}");
        sb.AppendLine($"  CPU clock : {perf.ClockText}{(perf.LikelyThrottled ? "  (possible throttling)" : "")}");
        if (s.Reliability.StabilityIndex is { } si) sb.AppendLine($"  Stability index: {si:F1} / 10");
        if (s.RebootPending) sb.AppendLine($"  Reboot pending : {string.Join(", ", s.RebootReasons)}");

        H("POWER & BATTERY");
        sb.AppendLine($"  Power source : {s.Power.PowerSource}");
        sb.AppendLine($"  Power mode   : {s.Power.PowerMode}");
        sb.AppendLine($"  Fast startup : {s.Power.FastStartup}");
        sb.AppendLine($"  Hibernation  : {s.Power.Hibernation}");
        sb.AppendLine($"  Sleep        : {s.Power.Sleep}");
        sb.AppendLine($"  Display off  : {s.Power.DisplayOff}");
        sb.AppendLine($"  Last wake    : {s.Power.LastWake}");

        H("SECURITY");
        int fwOff = new[] { sec.FwDomain, sec.FwPrivate, sec.FwPublic }.Count(v => !v);
        sb.AppendLine($"  Antivirus  : {sec.AvName} (RTP {(sec.AvRtpEnabled ? "on" : "off")})");
        sb.AppendLine($"  Firewall   : {(fwOff == 0 ? "all profiles on" : $"{fwOff} profile(s) off")}");
        sb.AppendLine($"  BitLocker  : {sec.BitLocker}");
        sb.AppendLine($"  UAC        : {(sec.UacEnabled == true ? "Enabled" : sec.UacEnabled == false ? "Disabled" : "Unknown")}");
        sb.AppendLine($"  Activation : {(sec.Activated == true ? "Activated" : sec.Activated == false ? "Not activated" : "Unknown")}");
        sb.AppendLine($"  Secure Boot: {(sec.SecureBoot == true ? "Enabled" : sec.SecureBoot == false ? "Disabled" : "Unknown")}");

        if (d.Hardening.Count > 0)
        {
            H("HARDENING SCORECARD");
            foreach (var it in d.Hardening) sb.AppendLine($"  [{it.Level,-4}] {it.Name} — {it.Detail}");
        }

        H("NETWORK — PRIMARY ADAPTER");
        if (s.PrimaryAdapter is { } a)
        {
            sb.AppendLine($"  Adapter   : {a.Name} ({a.Type})");
            sb.AppendLine($"  IP / Mask : {a.IpAddress} / {a.SubnetMask}  [{a.IpAssignment}]");
            sb.AppendLine($"  Gateway   : {a.Gateway}");
            sb.AppendLine($"  DNS       : {a.Dns}");
            sb.AppendLine($"  MAC       : {a.Mac}");
        }
        else sb.AppendLine("  No active adapter detected.");

        if (d.Adapters.Count > 0)
        {
            H("NETWORK — ALL ADAPTERS");
            foreach (var ad in d.Adapters)
                sb.AppendLine($"  {ad.Name} ({ad.Type}) — {ad.IpAddress} / {ad.SubnetMask} [{ad.IpAssignment}], GW {ad.Gateway}, DNS {ad.Dns}, MAC {ad.Mac}");
        }
        if (d.Wifi.Count > 0)
        {
            H("SAVED WI-FI NETWORKS");
            foreach (var w in d.Wifi) sb.AppendLine($"  • {w}");
        }

        if (s.Displays.Count > 0)
        {
            H("MONITORS");
            foreach (var m in s.Displays) sb.AppendLine($"  {m.Name} — {m.Detail}");
        }
        if (s.Printers.Count > 0)
        {
            H("PRINTERS");
            foreach (var p in s.Printers) sb.AppendLine($"  {p.Display} — {p.Detail}");
        }
        if (s.Accounts.Count > 0)
        {
            H("LOCAL ACCOUNTS");
            foreach (var ac in s.Accounts) sb.AppendLine($"  {ac.Name} — {ac.Detail}");
        }

        H($"INSTALLED SOFTWARE ({d.Software.Count})");
        foreach (var app in d.Software)
            sb.AppendLine($"  {app.Name}{(app.Version.Length > 0 ? $"  v{app.Version}" : "")}" +
                          $"{(app.Publisher.Length > 0 ? $"  · {app.Publisher}" : "")}" +
                          $"{(app.InstallDate.Length > 0 ? $"  · {app.InstallDate}" : "")}");

        if (d.Startup.Count > 0)
        {
            H($"STARTUP PROGRAMS ({d.Startup.Count})");
            foreach (var su in d.Startup)
                sb.AppendLine($"  [{su.StateText,-8}] {su.Name} — {su.LocationText}");
        }

        if (s.Updates.Recent.Count > 0)
        {
            H("WINDOWS UPDATE HISTORY (recent)");
            foreach (var u in s.Updates.Recent)
                sb.AppendLine($"  {u.Date:MM/dd/yyyy}  [{u.Result}]  {u.Title}");
        }

        if (s.Diagnostics.Devices.Count > 0)
        {
            H("DEVICE PROBLEMS");
            foreach (var dev in s.Diagnostics.Devices) sb.AppendLine($"  {dev.Name} — {dev.Problem}");
        }
        H("CRASH DUMPS");
        sb.AppendLine($"  Minidumps: {s.Diagnostics.MinidumpCount}" +
                      (s.Diagnostics.LatestDump is { } ld ? $" (latest {ld:MM/dd/yyyy})" : "") +
                      $"  ·  Full memory dump: {(s.Diagnostics.MemoryDump ? "present" : "none")}");

        H("SYSTEM EXTRAS");
        sb.AppendLine($"  Power plan : {s.Extras.PowerPlan}");
        sb.AppendLine($"  Page file  : {s.Extras.PageFile}");
        sb.AppendLine($"  Proxy      : {s.Extras.Proxy}");

        return sb.ToString();
    }

    // ──────────────────────────────────────────────────────────────────────
    //  HTML
    // ──────────────────────────────────────────────────────────────────────
    public static string BuildHtml(FullReportData d)
    {
        var s = d.Snap;
        var info = s.Info; var perf = s.Perf; var sec = s.Security; var hw = s.Hardware;
        static string E(string? v) => WebUtility.HtmlEncode(v ?? "");
        var sb = new StringBuilder();

        sb.Append($@"<!DOCTYPE html><html lang=""en""><head><meta charset=""utf-8"">
<title>Partner Tool — System Report — {E(info.Hostname)}</title>
<style>
 body{{background:#1E1E2E;color:#CDD6F4;font-family:Segoe UI,Arial,sans-serif;margin:0;padding:32px;}}
 h1{{color:#CBA6F7;font-size:22px;margin:0 0 4px;}}
 .sub{{color:#6C7086;font-size:12px;margin-bottom:18px;}}
 .toc{{background:#181825;border-radius:10px;padding:14px 18px;margin:0 0 18px;font-size:12px;line-height:2;}}
 .toc a{{color:#89B4FA;text-decoration:none;margin-right:16px;white-space:nowrap;}}
 .toc a:hover{{text-decoration:underline;}}
 .card{{background:#313244;border-radius:10px;padding:18px 22px;margin:0 0 16px;}}
 h2{{color:#CBA6F7;font-size:12px;letter-spacing:.5px;margin:0 0 12px;text-transform:uppercase;}}
 h2 .count{{color:#6C7086;font-weight:normal;}}
 table{{width:100%;border-collapse:collapse;}}
 th{{text-align:left;color:#6C7086;font-size:10px;text-transform:uppercase;letter-spacing:.5px;padding:4px 8px;border-bottom:1px solid #45475A;}}
 td{{padding:6px 8px;font-size:13px;border-bottom:1px solid #45475A;vertical-align:top;}}
 td.k{{color:#6C7086;width:220px;}}
 tr:last-child td{{border-bottom:none;}}
 .empty{{color:#6C7086;font-size:12px;}}
 .ok{{color:#A6E3A1;}} .warn{{color:#F9E2AF;}} .bad{{color:#F38BA8;}} .info{{color:#89B4FA;}}
</style></head><body>
<h1>Partner Support Tool — System Report</h1>
<div class=""sub"">Progressive Computing &middot; {E(info.Hostname)} &middot; generated {E(s.CapturedAt.ToString(Dates.DateTimeSec))}</div>");

        // Table of contents
        sb.Append(@"<div class=""toc"">
<a href=""#identity"">Identity</a><a href=""#hardware"">Hardware</a><a href=""#os"">Operating system</a>
<a href=""#perf"">Performance</a><a href=""#power"">Power</a><a href=""#security"">Security</a>
<a href=""#hardening"">Hardening</a><a href=""#network"">Network</a><a href=""#monitors"">Monitors</a>
<a href=""#printers"">Printers</a><a href=""#accounts"">Accounts</a><a href=""#software"">Software</a>
<a href=""#startup"">Startup</a><a href=""#updates"">Updates</a><a href=""#devices"">Device problems</a>
<a href=""#extras"">Extras</a></div>");

        // ── Local helpers ──
        void Card(string id, string title, params (string k, string v)[] rows)
        {
            sb.Append($"<div class=\"card\" id=\"{id}\"><h2>{E(title)}</h2><table>");
            foreach (var (k, v) in rows)
                sb.Append($"<tr><td class=\"k\">{E(k)}</td><td>{E(v)}</td></tr>");
            sb.Append("</table></div>");
        }

        void TableCard(string id, string title, string[] headers, List<string[]> rows, string emptyNote = "None.")
        {
            int count = rows.Count;
            sb.Append($"<div class=\"card\" id=\"{id}\"><h2>{E(title)} <span class=\"count\">({count})</span></h2>");
            if (count == 0) { sb.Append($"<div class=\"empty\">{E(emptyNote)}</div></div>"); return; }
            sb.Append("<table><thead><tr>");
            foreach (var h in headers) sb.Append($"<th>{E(h)}</th>");
            sb.Append("</tr></thead><tbody>");
            foreach (var r in rows)
            {
                sb.Append("<tr>");
                foreach (var c in r) sb.Append($"<td>{E(c)}</td>");
                sb.Append("</tr>");
            }
            sb.Append("</tbody></table></div>");
        }

        // ── Identity + device join ──
        Card("identity", "Identity",
            ("Hostname", info.Hostname), ("Logged in user", info.LoggedInUser),
            ("Domain", info.Domain), ("Timezone", info.Timezone),
            ("Azure AD joined", YesNo(s.Aad.AzureAdJoined)),
            ("Domain joined", YesNo(s.Aad.DomainJoined)),
            ("Enterprise joined", YesNo(s.Aad.EnterpriseJoined)),
            ("Tenant", s.Aad.TenantName), ("Device ID", s.Aad.DeviceId));

        // ── Hardware ──
        var hwRows = new List<(string, string)>
        {
            ("Manufacturer / Model", info.ManufacturerModel),
            ("Serial number", info.SerialNumber),
            ("Processor", $"{perf.CpuName} ({perf.Cores}C / {perf.Threads}T)"),
            ("CPU clock", perf.ClockText + (perf.LikelyThrottled ? " (possible throttling)" : "")),
            ("Memory", $"{hw.TotalMemoryGb:F0} GB ({hw.SlotsUsed}/{hw.SlotsTotal} slots, max {hw.MaxMemoryGb:F0} GB)"),
            ("BIOS", $"{hw.BiosVersion} ({hw.BiosDate}) · {hw.BootMode} · TPM {hw.Tpm}"),
        };
        if (hw.BatteryWearPct is { } bw) hwRows.Add(("Battery wear", $"{bw}% (design {hw.BatteryDesignMwh} mWh, full {hw.BatteryFullMwh} mWh)"));
        if (s.Temps.CpuTemp is { } ct) hwRows.Add(("CPU temperature", $"{ct:F0} °C"));
        if (s.Temps.GpuTemp is { } gt) hwRows.Add(("GPU temperature", $"{gt:F0} °C"));
        Card("hardware", "Hardware", hwRows.ToArray());

        TableCard("memory", "Memory modules", new[] { "Slot", "Size", "Speed", "Manufacturer" },
            hw.Memory.Select(m => new[] { m.Slot, $"{m.SizeGb:F0} GB", $"{m.SpeedMhz} MHz", m.Maker }).ToList(),
            "No per-module data.");

        TableCard("disks", "Disks", new[] { "Model", "Type", "Size", "Health", "SMART" },
            hw.Disks.Select(disk => new[] { disk.Model, disk.Type, $"{disk.SizeGb:F0} GB", disk.Health, disk.SmartText }).ToList());

        TableCard("volumes", "Volumes", new[] { "Drive", "Label", "File system", "Used", "Free", "Used %" },
            hw.Volumes.Select(v => new[] { v.Letter, v.Label, v.FileSystem, $"{v.UsedGb:F0} GB", $"{v.FreeGb:F0} GB", $"{v.UsedPct:F0}%" }).ToList());

        TableCard("graphics", "Graphics", new[] { "Adapter", "VRAM", "Driver", "Driver date", "Resolution" },
            hw.Gpus.Select(g => new[] { g.Name, g.VramGb > 0 ? $"{g.VramGb:F1} GB" : "—", g.Driver, g.DriverDate, g.Resolution }).ToList());

        // ── OS / performance / power ──
        Card("os", "Operating system",
            ("OS version", info.OsVersion), ("Feature version", info.WindowsVersion),
            ("Kernel", $"{info.KernelVersion} ({info.KernelArchitecture})"),
            ("Last patched", s.Updates.LastInstalled is { } lp ? lp.ToString(Dates.Date) : "Unknown"));

        Card("perf", "Performance",
            ("Memory", $"{perf.RamUsedGb:F1} / {perf.RamTotalGb:F1} GB ({perf.RamPct:F0}%)"),
            ("Disk C:", $"{perf.DiskUsedGb:F0} / {perf.DiskTotalGb:F0} GB ({perf.DiskPct:F0}%)"),
            ("Uptime", perf.UptimeText),
            ("Stability index", s.Reliability.StabilityIndex is { } si ? $"{si:F1} / 10" : "—"),
            ("Reboot pending", s.RebootPending ? string.Join(", ", s.RebootReasons) : "No"));

        Card("power", "Power & battery",
            ("Power source", s.Power.PowerSource), ("Power mode", s.Power.PowerMode),
            ("Fast startup", s.Power.FastStartup), ("Hibernation", s.Power.Hibernation),
            ("Sleep", s.Power.Sleep), ("Display off", s.Power.DisplayOff), ("Last wake", s.Power.LastWake));

        // ── Security ──
        int fwOff = new[] { sec.FwDomain, sec.FwPrivate, sec.FwPublic }.Count(v => !v);
        Card("security", "Security",
            ("Antivirus", $"{sec.AvName} (RTP {(sec.AvRtpEnabled ? "on" : "off")})"),
            ("Firewall", fwOff == 0 ? "All profiles on" : $"{fwOff} profile(s) off"),
            ("BitLocker", sec.BitLocker),
            ("UAC", sec.UacEnabled == true ? "Enabled" : sec.UacEnabled == false ? "Disabled" : "Unknown"),
            ("Activation", sec.Activated == true ? "Activated" : sec.Activated == false ? "Not activated" : "Unknown"),
            ("Secure Boot", sec.SecureBoot == true ? "Enabled" : sec.SecureBoot == false ? "Disabled" : "Unknown"));

        // Hardening scorecard (colored by level)
        sb.Append("<div class=\"card\" id=\"hardening\"><h2>Hardening scorecard</h2>");
        if (d.Hardening.Count == 0) sb.Append("<div class=\"empty\">Not collected.</div>");
        else
        {
            sb.Append("<table>");
            foreach (var it in d.Hardening)
            {
                string cls = it.Level switch
                {
                    AuditLevel.Good => "ok", AuditLevel.Warn => "warn",
                    AuditLevel.Bad  => "bad", _ => "info",
                };
                sb.Append($"<tr><td class=\"k\">{E(it.Name)}</td><td class=\"{cls}\">{E(it.Detail)}</td></tr>");
            }
            sb.Append("</table>");
        }
        sb.Append("</div>");

        // ── Network ──
        if (s.PrimaryAdapter is { } a)
            Card("network", "Network — primary adapter",
                ("Adapter", $"{a.Name} ({a.Type})"), ("IP / Mask", $"{a.IpAddress} / {a.SubnetMask}"),
                ("Assignment", a.IpAssignment), ("Gateway", a.Gateway), ("DNS", a.Dns), ("MAC", a.Mac));
        else
            Card("network", "Network — primary adapter", ("Status", "No active adapter detected."));

        TableCard("alladapters", "Network — all adapters", new[] { "Adapter", "Type", "IP / Mask", "Assignment", "Gateway", "DNS", "MAC" },
            d.Adapters.Select(x => new[] { x.Name, x.Type, $"{x.IpAddress} / {x.SubnetMask}", x.IpAssignment, x.Gateway, x.Dns, x.Mac }).ToList());

        TableCard("wifi", "Saved Wi-Fi networks", new[] { "Profile" },
            d.Wifi.Select(w => new[] { w }).ToList(), "No saved Wi-Fi profiles.");

        // ── Devices / accounts ──
        TableCard("monitors", "Monitors", new[] { "Display", "Details" },
            s.Displays.Select(m => new[] { m.Name, m.Detail }).ToList());

        TableCard("printers", "Printers", new[] { "Printer", "Details" },
            s.Printers.Select(p => new[] { p.Display, p.Detail }).ToList());

        TableCard("accounts", "Local accounts", new[] { "Account", "Details" },
            s.Accounts.Select(ac => new[] { ac.Name, ac.Detail }).ToList());

        // ── Software / startup ──
        TableCard("software", "Installed software", new[] { "Name", "Version", "Publisher", "Installed" },
            d.Software.Select(app => new[] { app.Name, app.Version, app.Publisher, app.InstallDate }).ToList());

        TableCard("startup", "Startup programs", new[] { "Name", "State", "Location", "Command" },
            d.Startup.Select(su => new[] { su.Name, su.StateText, su.LocationText, su.Command }).ToList());

        // ── Updates / devices / extras ──
        TableCard("updates", "Windows Update history", new[] { "Date", "Result", "Title" },
            s.Updates.Recent.Select(u => new[] { u.Date.ToString(Dates.Date), u.Result, u.Title }).ToList());

        TableCard("devices", "Device problems", new[] { "Device", "Problem" },
            s.Diagnostics.Devices.Select(dev => new[] { dev.Name, dev.Problem }).ToList(),
            "No devices reporting problems.");

        Card("extras", "System extras",
            ("Power plan", s.Extras.PowerPlan), ("Page file", s.Extras.PageFile), ("Proxy", s.Extras.Proxy),
            ("Crash dumps", $"{s.Diagnostics.MinidumpCount} minidump(s)" +
                (s.Diagnostics.LatestDump is { } ld ? $", latest {ld:MM/dd/yyyy}" : "") +
                $"; full memory dump {(s.Diagnostics.MemoryDump ? "present" : "none")}"));

        sb.Append("</body></html>");
        return sb.ToString();
    }

    private static string YesNo(bool v) => v ? "Yes" : "No";
}
```
