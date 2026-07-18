---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\FeatureUpdateCleanup.cs
---

# PartnerTool\FeatureUpdateCleanup.cs

```csharp
using System.IO;
using System.Text;

namespace PartnerTool;

/// <summary>One leftover location a Windows feature update can leave behind.</summary>
public class FeatureUpdateItem
{
    public required string Path      { get; init; }
    public required string Label     { get; init; }
    public required bool   Permanent { get; init; }   // true → deleting it loses something (e.g. rollback)
    public required string Note      { get; init; }   // shown in the preview/confirm
    public long   Bytes { get; set; }

    public double Gb => Bytes / 1073741824.0;
}

public class FeatureUpdateScan
{
    public readonly List<FeatureUpdateItem> Items = new();   // only the ones that actually exist
    public long TotalBytes => Items.Sum(i => i.Bytes);
    public double TotalGb  => TotalBytes / 1073741824.0;
    public bool AnyPermanent => Items.Any(i => i.Permanent);
}

/// <summary>
/// The space Windows Disk Cleanup reclaims that our temp-cleaner doesn't: the big feature-update
/// leftovers. Covers the previous-Windows rollback image and upgrade staging (`Windows.old`,
/// `$Windows.~BT`, `$Windows.~WS`, `$GetCurrent`) and the Delivery Optimization peer cache.
///
/// These are ACL-hardened (owned by TrustedInstaller/SYSTEM), so removing them needs an ownership
/// grab first — done in <see cref="BuildCleanScript"/> via takeown + icacls (Administrators by SID,
/// so it's locale-independent) then a recursive delete. Delivery Optimization is cleared with its
/// own cmdlet. Sizing is read-only and never follows reparse points.
///
/// ⚠ `Windows.old` &amp; the `$Windows.~*` staging are what Windows uses to ROLL BACK a feature
/// update (the ~10-day "go back" option). Deleting them is permanent and gives that up — the caller
/// tech-gates and confirms, spelling that out.
/// </summary>
public static class FeatureUpdateCleanup
{
    private const string DeliveryOptimizationPath = @"C:\Windows\SoftwareDistribution\DeliveryOptimization";

    // Candidate locations, in the order they're shown. label / permanent / note.
    private static readonly (string Path, string Label, bool Permanent, string Note)[] Targets =
    {
        (@"C:\Windows.old",   "Previous Windows installation (Windows.old)", true,
            "Used to roll back to the previous Windows build — removing it ends the “go back” option."),
        (@"C:\$Windows.~BT",  "Feature-update staging ($Windows.~BT)", true,
            "Leftover setup files from the last feature update."),
        (@"C:\$Windows.~WS",  "Feature-update download ($Windows.~WS)", true,
            "Downloaded feature-update media."),
        (@"C:\$GetCurrent",   "Feature-update logs ($GetCurrent)", false,
            "Upgrade log files; safe to remove."),
        (DeliveryOptimizationPath, "Delivery Optimization cache", false,
            "Peer-to-peer update cache; Windows rebuilds it as needed."),
    };

    /// <summary>Size every leftover that's actually present (read-only; nothing is deleted).</summary>
    public static FeatureUpdateScan Scan()
    {
        var scan = new FeatureUpdateScan();
        foreach (var (path, label, permanent, note) in Targets)
        {
            if (!Directory.Exists(path)) continue;
            var item = new FeatureUpdateItem { Path = path, Label = label, Permanent = permanent, Note = note };
            try { item.Bytes = DirSize(path); } catch { }
            scan.Items.Add(item);
        }
        return scan;
    }

    /// <summary>
    /// PowerShell that removes exactly the folders passed in (so the log only mentions what was
    /// really there). Each ACL-hardened folder gets takeown + icacls before the delete; Delivery
    /// Optimization is cleared with its own cmdlet, with a folder-delete fallback.
    /// </summary>
    public static string BuildCleanScript(IEnumerable<FeatureUpdateItem> items)
    {
        var sb = new StringBuilder();
        sb.AppendLine("$ErrorActionPreference = 'SilentlyContinue'");
        foreach (var item in items)
        {
            if (string.Equals(item.Path, DeliveryOptimizationPath, StringComparison.OrdinalIgnoreCase))
            {
                sb.AppendLine("Write-Output 'Clearing Delivery Optimization cache...'");
                sb.AppendLine("try { Delete-DeliveryOptimizationCache -Force -ErrorAction Stop; Write-Output '  cleared (cmdlet).' }");
                sb.AppendLine("catch {");
                // Fallback: empty the folder's CONTENTS (keep the folder itself). Enumerate then
                // delete — NOT `Remove-Item -LiteralPath '…\*'`, which treats * literally and no-ops.
                sb.AppendLine($"  Get-ChildItem -LiteralPath '{item.Path}' -Force -ErrorAction SilentlyContinue | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue");
                sb.AppendLine("  Write-Output '  cleared (folder).' }");
                continue;
            }

            // ACL-hardened upgrade leftovers: take ownership, grant Administrators (by SID, so it
            // works on non-English Windows), then delete recursively.
            sb.AppendLine($"$p = '{item.Path}'");
            sb.AppendLine("if (Test-Path -LiteralPath $p) {");
            sb.AppendLine($"  Write-Output 'Removing {item.Label}...'");
            sb.AppendLine("  takeown /f $p /r /d y | Out-Null");
            sb.AppendLine("  icacls $p /grant *S-1-5-32-544:F /t /c /q | Out-Null");
            sb.AppendLine("  Remove-Item -LiteralPath $p -Recurse -Force");
            sb.AppendLine("  if (Test-Path -LiteralPath $p) { Write-Output '  some items remained (in use) — a reboot may finish it.' }");
            sb.AppendLine("  else { Write-Output '  removed.' }");
            sb.AppendLine("}");
        }
        sb.AppendLine("Write-Output 'Feature-update cleanup complete.'");
        return sb.ToString();
    }

    private const int MaxDepth = 40;

    private static long DirSize(string path, int depth = 0)
    {
        if (depth > MaxDepth) return 0;
        long total = 0;
        try
        {
            var di = new DirectoryInfo(path);
            if ((di.Attributes & FileAttributes.ReparsePoint) != 0) return 0;   // don't follow junctions

            foreach (var f in di.EnumerateFiles())
                try { total += f.Length; } catch { }

            foreach (var d in di.EnumerateDirectories())
            {
                try
                {
                    if ((d.Attributes & FileAttributes.ReparsePoint) != 0) continue;
                    total += DirSize(d.FullName, depth + 1);
                }
                catch { }
            }
        }
        catch { }
        return total;
    }
}
```
