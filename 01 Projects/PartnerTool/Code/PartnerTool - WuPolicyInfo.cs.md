---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\WuPolicyInfo.cs
---

# PartnerTool\WuPolicyInfo.cs

```csharp
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>One Windows Update policy setting, for the Updates-tab source card.
/// <paramref name="Level"/> tints the row: Info (neutral), Warn (worth a look — a policy that can
/// cause "couldn't connect to the update service").</summary>
public record WuPolicyRow(string Name, string Value, AuditLevel Level = AuditLevel.Info);

/// <summary>
/// Where this PC gets Windows Updates, and the policies steering it — so a tech can see at a glance
/// whether a managed device is pointed at a (possibly dead) WSUS server or blocked from Microsoft's
/// servers, which is the usual reason for "we couldn't connect to the update service". All read-only
/// registry reads of the WindowsUpdate policy keys + the MDM PolicyManager key.
/// </summary>
public class WuPolicyInfo
{
    public string Source { get; init; } = "Windows Update (Microsoft) — not policy-managed";
    public bool   Managed { get; init; }
    public List<WuPolicyRow> Rows { get; init; } = new();

    private const string WuKey = @"SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate";
    private const string AuKey = @"SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU";
    private const string MdmKey = @"SOFTWARE\Microsoft\PolicyManager\current\device\Update";

    public static WuPolicyInfo Collect()
    {
        var rows = new List<WuPolicyRow>();
        string source = "Windows Update (Microsoft) — not policy-managed";
        bool managed = false;

        try
        {
            using var wu = Registry.LocalMachine.OpenSubKey(WuKey);
            using var au = Registry.LocalMachine.OpenSubKey(AuKey);
            using var mdm = Registry.LocalMachine.OpenSubKey(MdmKey);

            string? wuServer     = wu?.GetValue("WUServer") as string;
            string? statusServer = wu?.GetValue("WUStatusServer") as string;
            bool useWuServer     = Dword(au, "UseWUServer") == 1;

            // ── Where updates come from ──
            if (!string.IsNullOrWhiteSpace(wuServer) && useWuServer)
            {
                source  = $"WSUS server — {wuServer}";
                managed = true;
                rows.Add(new("WSUS server", wuServer!, AuditLevel.Warn));
                if (!string.IsNullOrWhiteSpace(statusServer) &&
                    !string.Equals(statusServer, wuServer, StringComparison.OrdinalIgnoreCase))
                    rows.Add(new("WSUS status server", statusServer!));
            }
            else if (mdm is { } m && m.GetValueNames().Length + m.GetSubKeyNames().Length > 0)
            {
                source  = "Managed by MDM / Intune (Update policies)";
                managed = true;
            }
            else if (wu != null || au != null)
            {
                source  = "Managed by Group Policy (Windows Update policies set)";
                managed = true;
            }

            // ── The policy most likely to cause "couldn't connect to the update service" ──
            if (Dword(wu, "DoNotConnectToWindowsUpdateInternetLocations") == 1)
                rows.Add(new("Reach Microsoft's servers",
                    "Blocked by policy — WSUS/Intune only. If that source is down, WU can't connect.",
                    AuditLevel.Warn));

            // ── Behaviour / access policies worth showing ──
            if (Dword(au, "NoAutoUpdate") is { } nau)
                rows.Add(new("Automatic updates", nau == 1 ? "Disabled by policy" : "Enabled",
                    nau == 1 ? AuditLevel.Warn : AuditLevel.Info));
            if (Dword(au, "AUOptions") is { } auo)
                rows.Add(new("Update behaviour", AuOptionsText(auo)));
            if (Dword(wu, "DisableWindowsUpdateAccess") == 1)
                rows.Add(new("Windows Update UI", "Access disabled by policy", AuditLevel.Warn));
            if (wu?.GetValue("TargetReleaseVersionInfo") as string is { Length: > 0 } trv)
                rows.Add(new("Pinned feature version", trv));
            if (Dword(wu, "DeferFeatureUpdatesPeriodInDays") is { } dfu && dfu > 0)
                rows.Add(new("Feature updates deferred", $"{dfu} day(s)"));
            if (Dword(wu, "DeferQualityUpdatesPeriodInDays") is { } dqu && dqu > 0)
                rows.Add(new("Quality updates deferred", $"{dqu} day(s)"));
        }
        catch { }

        return new WuPolicyInfo { Source = source, Managed = managed, Rows = rows };
    }

    private static int? Dword(RegistryKey? key, string name)
        => key?.GetValue(name) is int i ? i : null;

    private static string AuOptionsText(int o) => o switch
    {
        2 => "Notify before download",
        3 => "Auto-download, notify to install",
        4 => "Auto-download and schedule install",
        5 => "Local admin chooses",
        7 => "Auto-download, notify to install (no auto-reboot)",
        _ => o.ToString(),
    };
}
```
