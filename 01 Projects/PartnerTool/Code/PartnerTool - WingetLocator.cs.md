---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\WingetLocator.cs
---

# PartnerTool\WingetLocator.cs

```csharp
using System.Diagnostics;
using System.IO;

namespace PartnerTool;

/// <summary>
/// One reliable way to find a winget.exe that will actually RUN, used by every caller (Software
/// install, Updates "Update All", the outdated-apps scan and the vendor-tool installer).
///
/// Why this is hard: winget ships as the MSIX package Microsoft.DesktopAppInstaller. The familiar
/// `%LOCALAPPDATA%\Microsoft\WindowsApps\winget.exe` is a per-user App-Execution-Alias, and the real
/// binary under `C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_…` only grants **execute**
/// to accounts the package is REGISTERED for. PartnerTool runs elevated, very often as a throwaway
/// admin (LAPS/RMM accounts like `~0000AEAdmin`) that has never had Store apps provisioned. For that
/// account there is no alias, winget isn't on PATH, and the package path — which very much exists —
/// fails to launch with **"Access is denied."** (Field log, 0.19.6, COMMAI-PC-001.)
///
/// So: existence is not executability. Every candidate is probed by actually running `--version`, and
/// if none work we register the package for the current user (the supported fix), which creates the
/// alias and grants execute, then probe again.
///
/// Resolution is cached and <see cref="Warm"/> is called once at startup so the (slow) lookup never
/// blocks a UI action.
/// </summary>
public static class WingetLocator
{
    private const string FamilyName = "Microsoft.DesktopAppInstaller_8wekyb3d8bbwe";

    private static readonly object Gate = new();
    private static string? _cached;

    /// <summary>Kick off resolution on a background thread at startup so the cache is warm early.</summary>
    public static void Warm() => Task.Run(() => Path());

    /// <summary>Absolute path to a winget.exe that runs, or "" when winget is unusable here.</summary>
    public static string Path()
    {
        lock (Gate)
        {
            return _cached ??= Resolve();
        }
    }

    /// <summary>True when we found a winget.exe that actually launches for this account.</summary>
    public static bool IsAvailable() => Path().Length > 0;

    /// <summary>Why winget can't be used, for the UI to show instead of a raw "Access is denied".</summary>
    public const string Unavailable =
        "winget (App Installer) isn't available to this account. It's an MSIX app, and the elevated " +
        "account you're signed in as has never had it registered — so Windows refuses to launch it. " +
        "Sign in as the regular user, or install App Installer from the Microsoft Store.";

    private static string Resolve()
    {
        // 1. Per-user execution alias — the happy path when we're elevated AS a user who has winget.
        var alias = System.IO.Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            @"Microsoft\WindowsApps\winget.exe");
        if (Probe(alias)) return alias;

        // 2. The machine-wide package binary. Note WindowsApps denies directory LISTING to admins on
        // some builds while allowing traverse, so try enumeration but don't depend on it.
        var pkg = PackageExe();
        if (pkg != null && Probe(pkg)) return pkg;

        // 3. Nothing runs. Almost always because the package isn't registered for this elevated
        // account — register it, which creates the alias and grants execute, then retry.
        if (RegisterForCurrentUser())
        {
            if (Probe(alias)) return alias;
            pkg ??= PackageExe();
            if (pkg != null && Probe(pkg)) return pkg;
        }

        // 4. Let the OS try PATH, in case a portable winget is installed somewhere we don't know.
        if (Probe("winget.exe")) return "winget.exe";

        ActivityLog.Result("winget", "not usable by this account — see Unavailable message");
        return "";
    }

    /// <summary>
    /// Actually launch the candidate. An MSIX binary the current account isn't registered for exists
    /// on disk but throws "Access is denied" from CreateProcess, so File.Exists proves nothing.
    /// </summary>
    private static bool Probe(string exe)
    {
        try
        {
            var psi = new ProcessStartInfo(exe, "--version")
            {
                UseShellExecute        = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow         = true,
            };
            using var p = Process.Start(psi);
            if (p is null) return false;
            if (!p.WaitForExit(8000)) { try { p.Kill(true); } catch { } return false; }
            return p.ExitCode == 0;
        }
        catch { return false; }   // Win32Exception: "Access is denied" / "cannot find the file specified"
    }

    /// <summary>Path to winget.exe inside the installed package, by enumeration then by Get-AppxPackage.</summary>
    private static string? PackageExe()
    {
        foreach (var pf in new[]
                 {
                     Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles),
                     Environment.GetEnvironmentVariable("ProgramW6432") ?? "",
                 })
        {
            if (pf.Length == 0) continue;
            try
            {
                var apps = new DirectoryInfo(System.IO.Path.Combine(pf, "WindowsApps"));
                if (!apps.Exists) continue;
                var match = apps.GetDirectories("Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe")
                    .OrderByDescending(d => d.Name)
                    .Select(d => System.IO.Path.Combine(d.FullName, "winget.exe"))
                    .FirstOrDefault(File.Exists);
                if (match != null) return match;
            }
            catch { /* listing denied — fall through to the appx query */ }
        }

        // No listing needed: ask Windows for the package path directly. -AllUsers finds it even when
        // it was never registered for the elevated account (needs admin, which we have).
        var dir = PowerShellLine(
            "(Get-AppxPackage -AllUsers Microsoft.DesktopAppInstaller | " +
            "Sort-Object Version | Select-Object -Last 1).InstallLocation");
        if (dir.Length == 0) return null;
        var exe = System.IO.Path.Combine(dir, "winget.exe");
        return File.Exists(exe) ? exe : null;
    }

    /// <summary>
    /// Register the already-installed package for the account we're running as. This is the supported
    /// fix for "winget is missing for my admin account": it doesn't download or install anything, it
    /// just makes the existing machine-wide package usable by this user (creating the alias and the
    /// execute rights we were denied).
    /// </summary>
    private static bool RegisterForCurrentUser()
    {
        ActivityLog.Action("winget", $"Registering {FamilyName} for {Environment.UserName} (winget not usable)");
        var err = PowerShellLine(
            $"try {{ Add-AppxPackage -RegisterByFamilyName -MainPackage {FamilyName} -ErrorAction Stop; 'ok' }} " +
            "catch { 'fail: ' + $_.Exception.Message }");
        bool ok = err.StartsWith("ok", StringComparison.OrdinalIgnoreCase);
        ActivityLog.Result("winget", ok ? "registered" : (err.Length > 0 ? err : "registration failed"));
        return ok;
    }

    /// <summary>Run a one-liner in PowerShell and return its first line of output (trimmed).</summary>
    private static string PowerShellLine(string command)
    {
        try
        {
            var psi = new ProcessStartInfo(
                ProcessRunner.ResolveSystemExe("powershell.exe"),
                $"-NoProfile -NonInteractive -ExecutionPolicy Bypass -Command \"{command}\"")
            {
                UseShellExecute        = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow         = true,
            };
            using var p = Process.Start(psi);
            if (p is null) return "";
            string outp = p.StandardOutput.ReadToEnd();
            if (!p.WaitForExit(60000)) { try { p.Kill(true); } catch { } return ""; }
            return outp.Trim();
        }
        catch { return ""; }
    }
}
```
