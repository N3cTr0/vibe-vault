---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\StartupInfo.cs
---

# PartnerTool\StartupInfo.cs

```csharp
using System.IO;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>Where a startup entry lives — also tells us how to enable/disable it.</summary>
public enum StartupSource { RunHKLM, RunHKLM32, RunHKCU, FolderCommon, FolderUser }

public record StartupEntry(string Name, string Command, bool Enabled, StartupSource Source)
{
    public string LocationText => Source switch
    {
        StartupSource.RunHKLM      => "All users (registry)",
        StartupSource.RunHKLM32    => "All users (registry, 32-bit)",
        StartupSource.RunHKCU      => "Current user (registry)",
        StartupSource.FolderCommon => "All users (Startup folder)",
        StartupSource.FolderUser   => "Current user (Startup folder)",
        _ => "",
    };

    public string ToggleText => Enabled ? "Disable" : "Enable";
    public string StateText  => Enabled ? "Enabled" : "Disabled";
}

/// <summary>
/// Programs that launch at sign-in — from the Run registry keys and the Startup folders —
/// with their enabled/disabled state mirrored from the StartupApproved keys (the same
/// mechanism Task Manager's Startup tab uses). Supports toggling entries on/off.
/// </summary>
public static class StartupInfo
{
    private const string RunPath = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Run";
    private const string Run32   = @"SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Run";
    private const string ApprovedRun    = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\StartupApproved\Run";
    private const string ApprovedRun32  = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\StartupApproved\Run32";
    private const string ApprovedFolder = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\StartupApproved\StartupFolder";

    public static List<StartupEntry> Collect()
    {
        var list = new List<StartupEntry>();

        ScanRun(Registry.LocalMachine, RunPath, ApprovedRun,   StartupSource.RunHKLM,   list);
        ScanRun(Registry.LocalMachine, Run32,   ApprovedRun32, StartupSource.RunHKLM32, list);
        ScanRun(Registry.CurrentUser,  RunPath, ApprovedRun,   StartupSource.RunHKCU,   list);

        ScanFolder(Environment.GetFolderPath(Environment.SpecialFolder.CommonStartup),
                   Registry.LocalMachine, StartupSource.FolderCommon, list);
        ScanFolder(Environment.GetFolderPath(Environment.SpecialFolder.Startup),
                   Registry.CurrentUser, StartupSource.FolderUser, list);

        return list.OrderBy(e => e.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    private static void ScanRun(RegistryKey root, string runPath, string approvedPath,
                                StartupSource src, List<StartupEntry> list)
    {
        try
        {
            using var run = root.OpenSubKey(runPath);
            if (run == null) return;
            using var approved = root.OpenSubKey(approvedPath);
            foreach (var name in run.GetValueNames())
            {
                var cmd = run.GetValue(name)?.ToString() ?? "";
                bool enabled = IsApproved(approved?.GetValue(name) as byte[]);
                list.Add(new StartupEntry(name, cmd, enabled, src));
            }
        }
        catch { }
    }

    private static void ScanFolder(string folder, RegistryKey approvedRoot,
                                   StartupSource src, List<StartupEntry> list)
    {
        try
        {
            if (!Directory.Exists(folder)) return;
            using var approved = approvedRoot.OpenSubKey(ApprovedFolder);
            foreach (var file in Directory.GetFiles(folder))
            {
                var name = Path.GetFileName(file);
                if (name.Equals("desktop.ini", StringComparison.OrdinalIgnoreCase)) continue;
                bool enabled = IsApproved(approved?.GetValue(name) as byte[]);
                list.Add(new StartupEntry(Path.GetFileNameWithoutExtension(file), file, enabled, src));
            }
        }
        catch { }
    }

    // StartupApproved stores a 12-byte blob; an even first byte (0x02) = enabled,
    // an odd/0x03 first byte = disabled. A missing value means enabled.
    private static bool IsApproved(byte[]? blob) => blob is not { Length: > 0 } || (blob[0] & 1) == 0;

    /// <summary>True for startup entries belonging to AV/EDR/security software — disabling those
    /// would weaken the machine's protection, so the UI refuses. Uses the shared
    /// <see cref="SecuritySoftware"/> matcher against the entry name + command line.</summary>
    public static bool IsSecurityCritical(StartupEntry entry)
        => SecuritySoftware.Matches(entry.Name + " " + entry.Command);

    /// <summary>Enable or disable an entry by writing its StartupApproved value.</summary>
    public static void SetEnabled(StartupEntry entry, bool enable)
    {
        // Backstop for the UI check: never disable a security product's startup entry.
        if (!enable && IsSecurityCritical(entry))
            throw new InvalidOperationException(
                "This is a security product's startup entry — disabling it is blocked.");

        var (root, approvedPath, valueName) = entry.Source switch
        {
            StartupSource.RunHKLM      => (Registry.LocalMachine, ApprovedRun,    entry.Name),
            StartupSource.RunHKLM32    => (Registry.LocalMachine, ApprovedRun32,  entry.Name),
            StartupSource.RunHKCU      => (Registry.CurrentUser,  ApprovedRun,    entry.Name),
            StartupSource.FolderCommon => (Registry.LocalMachine, ApprovedFolder, Path.GetFileName(entry.Command)),
            StartupSource.FolderUser   => (Registry.CurrentUser,  ApprovedFolder, Path.GetFileName(entry.Command)),
            _ => (Registry.CurrentUser, ApprovedRun, entry.Name),
        };

        using var key = root.CreateSubKey(approvedPath, writable: true);
        var blob = new byte[12];
        blob[0] = (byte)(enable ? 0x02 : 0x03);
        if (!enable)
        {
            // Task Manager stamps the disable time into bytes 4–11; harmless if we do too.
            var ft = BitConverter.GetBytes(DateTime.UtcNow.ToFileTimeUtc());
            Array.Copy(ft, 0, blob, 4, 8);
        }
        key.SetValue(valueName, blob, RegistryValueKind.Binary);
    }
}
```
