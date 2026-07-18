---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TempCleaner.cs
---

# PartnerTool\TempCleaner.cs

```csharp
using System.IO;

namespace PartnerTool;

public class TempCleanResult
{
    public long BytesFreed;
    public int  FilesDeleted;
    public int  Skipped;
    public int  ProfilesProcessed;
    public readonly List<string> SkippedFiles = new();   // in-use files left behind (for the log)

    public string Summary =>
        $"Freed {BytesFreed / 1048576.0:F0} MB — {FilesDeleted:N0} file(s) across {ProfilesProcessed} profile(s)"
        + (Skipped > 0 ? $"  ·  {Skipped:N0} in use/skipped." : ".");
}

/// <summary>Result of a dry-run scan: what Clean *would* free, with a per-folder breakdown.</summary>
public class TempScanResult
{
    public long TotalBytes;
    public int  FileCount;
    public int  ProfilesProcessed;
    public readonly List<string> Lines = new();   // per-folder breakdown for the log

    public string Summary =>
        $"{TotalBytes / 1048576.0:F0} MB in {FileCount:N0} file(s) across {ProfilesProcessed} profile(s) can be cleaned.";
}

/// <summary>
/// Deletes the contents of the approved temp/cache folders across every user profile, plus the
/// system temp. Keeps the folders themselves; skips locked/in-use files; and refuses to follow
/// directory junctions/symlinks so a planted reparse point can't redirect this elevated delete
/// outside temp. Scope (user-approved): per-user Temp, CrashDumps, Windows Error Reporting,
/// INetCache, D3DSCache, RDP client cache, and C:\Windows\Temp. No browser/Office caches.
/// </summary>
public static class TempCleaner
{
    private static readonly string[] ProfileTargets =
    {
        @"AppData\Local\Temp",
        @"AppData\Local\CrashDumps",
        @"AppData\Local\Microsoft\Windows\WER",
        @"AppData\Local\Microsoft\Windows\INetCache",
        @"AppData\Local\D3DSCache",
        @"AppData\Local\Microsoft\Terminal Server Client\Cache",
    };

    // When shipped single-file, the exe self-extracts its native DLLs to %TEMP%\.net\PartnerTool-<ver>\…
    // and holds them open while running — so the temp cleaner would only ever "skip (in use)" them and
    // log noise. Skip our own extraction folder outright. Null in a normal (multi-file) dev build.
    private static readonly string? SelfExtractDir = FindSelfExtractDir();

    private static string? FindSelfExtractDir()
    {
        try
        {
            var baseDir = AppContext.BaseDirectory.TrimEnd('\\');
            // Only relevant when we're actually running from the single-file temp extraction path.
            return baseDir.Contains(@"\.net\", StringComparison.OrdinalIgnoreCase)
                ? Directory.GetParent(baseDir)?.FullName   // the "PartnerTool-<ver>" folder, above the hash
                : null;
        }
        catch { return null; }
    }

    private static bool IsSelfExtract(string dir) =>
        SelfExtractDir != null &&
        (string.Equals(dir, SelfExtractDir, StringComparison.OrdinalIgnoreCase) ||
         dir.StartsWith(SelfExtractDir + "\\", StringComparison.OrdinalIgnoreCase));

    public static TempCleanResult Clean(Action<string>? onStatus = null, Action<string>? onLog = null)
    {
        var r = new TempCleanResult();
        try
        {
            foreach (var profile in Directory.GetDirectories(@"C:\Users"))
            {
                var name = Path.GetFileName(profile);
                if (name is "Public" or "Default" or "Default User" or "All Users") continue;
                onStatus?.Invoke($"Cleaning {name}…");
                onLog?.Invoke($"User: {name}");

                bool touched = false;
                foreach (var rel in ProfileTargets)
                {
                    var dir = Path.Combine(profile, rel);
                    if (Directory.Exists(dir)) { CleanFolderLogged(dir, r, onLog); touched = true; }
                }
                if (touched) r.ProfilesProcessed++;
            }
        }
        catch { }

        onStatus?.Invoke("Cleaning system temp…");
        onLog?.Invoke("System:");
        CleanFolderLogged(@"C:\Windows\Temp", r, onLog);

        if (r.SkippedFiles.Count > 0)
        {
            onLog?.Invoke($"In use / skipped ({r.SkippedFiles.Count}):");
            foreach (var sf in r.SkippedFiles.Take(50)) onLog?.Invoke($"    • {sf}");
            if (r.SkippedFiles.Count > 50) onLog?.Invoke($"    …and {r.SkippedFiles.Count - 50:N0} more");
        }
        return r;
    }

    /// <summary>Dry-run: measure how much the same targets hold, WITHOUT deleting anything.</summary>
    public static TempScanResult Scan(Action<string>? onStatus = null)
    {
        var r = new TempScanResult();
        try
        {
            foreach (var profile in Directory.GetDirectories(@"C:\Users"))
            {
                var name = Path.GetFileName(profile);
                if (name is "Public" or "Default" or "Default User" or "All Users") continue;
                onStatus?.Invoke($"Scanning {name}…");

                bool touched = false;
                foreach (var rel in ProfileTargets)
                {
                    var dir = Path.Combine(profile, rel);
                    if (Directory.Exists(dir)) { ScanFolder(dir, r); touched = true; }
                }
                if (touched) r.ProfilesProcessed++;
            }
        }
        catch { }

        onStatus?.Invoke("Scanning system temp…");
        ScanFolder(@"C:\Windows\Temp", r);
        return r;
    }

    private static void ScanFolder(string dir, TempScanResult r)
    {
        long b0 = r.TotalBytes; int f0 = r.FileCount;
        SizeContents(dir, r);
        int df = r.FileCount - f0;
        long mb = (r.TotalBytes - b0) / 1048576;
        if (df > 0) r.Lines.Add($"  {dir} — {df:N0} file(s), {mb:N0} MB");
    }

    private static void SizeContents(string dir, TempScanResult r)
    {
        if (IsSelfExtract(dir)) return;   // our own in-use extracted DLLs — never count/clean them
        DirectoryInfo di;
        try
        {
            di = new DirectoryInfo(dir);
            if (!di.Exists) return;
            if ((di.Attributes & FileAttributes.ReparsePoint) != 0) return;   // don't follow junctions
        }
        catch { return; }

        FileInfo[] files;
        try { files = di.GetFiles(); } catch { return; }
        foreach (var f in files)
        {
            try { r.TotalBytes += f.Length; r.FileCount++; } catch { }
        }

        DirectoryInfo[] subs;
        try { subs = di.GetDirectories(); } catch { return; }
        foreach (var sub in subs)
        {
            try { if ((sub.Attributes & FileAttributes.ReparsePoint) != 0) continue; } catch { continue; }
            SizeContents(sub.FullName, r);
        }
    }

    // Clean one top-level target folder and log a one-line per-folder breakdown.
    private static void CleanFolderLogged(string dir, TempCleanResult r, Action<string>? onLog)
    {
        long b0 = r.BytesFreed; int f0 = r.FilesDeleted, s0 = r.Skipped;
        CleanContents(dir, r);
        int df = r.FilesDeleted - f0, sk = r.Skipped - s0;
        long mb = (r.BytesFreed - b0) / 1048576;
        if (df > 0 || sk > 0)
            onLog?.Invoke($"  {dir} — {df:N0} file(s), {mb:N0} MB" + (sk > 0 ? $", {sk} skipped" : ""));
    }

    private static void CleanContents(string dir, TempCleanResult r)
    {
        if (IsSelfExtract(dir)) return;   // our own in-use extracted DLLs — never try to delete them
        DirectoryInfo di;
        try
        {
            di = new DirectoryInfo(dir);
            if (!di.Exists) return;
            if ((di.Attributes & FileAttributes.ReparsePoint) != 0) return;   // never follow a junction/symlink
        }
        catch { return; }

        FileInfo[] files;
        try { files = di.GetFiles(); } catch { return; }
        foreach (var f in files)
        {
            try
            {
                long len = f.Length;
                if (f.IsReadOnly) f.IsReadOnly = false;
                f.Delete();
                r.BytesFreed += len;
                r.FilesDeleted++;
            }
            catch { r.Skipped++; r.SkippedFiles.Add(f.FullName); }   // locked / in use — leave it
        }

        DirectoryInfo[] subs;
        try { subs = di.GetDirectories(); } catch { return; }
        foreach (var sub in subs)
        {
            try { if ((sub.Attributes & FileAttributes.ReparsePoint) != 0) continue; } catch { continue; }
            CleanContents(sub.FullName, r);
            try { sub.Delete(false); } catch { }   // remove if it's now empty
        }
    }
}
```
