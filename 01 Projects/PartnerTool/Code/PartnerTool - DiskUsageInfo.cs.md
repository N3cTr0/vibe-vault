---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DiskUsageInfo.cs
---

# PartnerTool\DiskUsageInfo.cs

```csharp
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

namespace PartnerTool;

/// <summary>
/// One row in the disk-usage explorer — a sub-folder (with its recursive size) or a file.
/// NOTE: these are PROPERTIES, not fields — WPF data binding only sees properties.
/// </summary>
public class UsageEntry
{
    public required string Name     { get; init; }
    public required string FullPath { get; init; }
    public long   Bytes  { get; init; }
    public bool   IsDir  { get; init; }
    public int    Items  { get; init; }   // files under a folder (recursive); 0 for a file
    public bool   Sized  { get; init; } = true;   // false while a folder's size is still being computed
    public uint   Attributes { get; init; }       // raw FileAttributes (Hidden / System / ReadOnly / …)
    public DateTime Modified { get; init; }
    public bool   IsParent { get; init; }         // synthetic ".." row that navigates up one level
    public double Percent { get; set; }           // share of the folder total (set after sizing)

    public bool   Hidden => (Attributes & 0x06) != 0;   // FILE_ATTRIBUTE_HIDDEN (0x02) | SYSTEM (0x04)

    // ── WizTree-style columns ──
    public string Display      => IsParent ? ".. (up one level)" : Name;
    public string PercentText  => IsParent ? "" : $"{Percent:F0}%";
    public string SizeText     => IsParent ? "" : (Sized ? FormatSize(Bytes) : "…");
    public string FilesText    => (IsParent || !IsDir) ? "" : (Sized ? $"{Items:N0}" : "…");
    public string ModifiedText => (IsParent || Modified == default) ? "" : Modified.ToString(Dates.DateTime);
    public string AttrText     => IsParent ? "" : AttrString();

    /// <summary>Label for UI Automation so screen readers announce the row, not "PartnerTool.UsageEntry".</summary>
    public string AutomationName => IsParent
        ? "Up one level"
        : $"{(IsDir ? "Folder" : "File")} {Name}, {SizeText}";

    private string AttrString()
    {
        var a = (FileAttributes)Attributes;
        var s = "";
        if ((a & FileAttributes.ReadOnly)   != 0) s += "R";
        if ((a & FileAttributes.Hidden)     != 0) s += "H";
        if ((a & FileAttributes.System)     != 0) s += "S";
        if ((a & FileAttributes.Compressed) != 0) s += "C";
        return s;
    }

    public static string FormatSize(long b) =>
        b >= 1073741824 ? $"{b / 1073741824.0:F1} GB" :
        b >= 1048576    ? $"{b / 1048576.0:F0} MB"    :
        b >= 1024       ? $"{b / 1024.0:F0} KB"       : $"{b} B";
}

/// <summary>A fixed (internal hard) disk the user can scan.</summary>
public record DriveEntry(string Root, string Label, long TotalBytes, long FreeBytes)
{
    public string Display => $"{Root.TrimEnd('\\')}  {Label}  ·  " +
                             $"{UsageEntry.FormatSize(TotalBytes - FreeBytes)} / {UsageEntry.FormatSize(TotalBytes)}";
}

/// <summary>
/// "What's eating the drive?" — a TreeSize-style drill-down. Pick a fixed drive, then step into any
/// folder. <see cref="QuickList"/> shows the folder/file names instantly (folders A→Z); <see
/// cref="Children"/> then computes each folder's recursive size in parallel and re-sorts
/// largest-first. Reparse points (junctions/symlinks) are skipped; the walk honors cancellation.
/// </summary>
public static class DiskUsageInfo
{
    /// <summary>Fixed (internal hard) disks only — excludes network, removable and optical drives.</summary>
    public static List<DriveEntry> FixedDrives()
    {
        var list = new List<DriveEntry>();
        foreach (var d in DriveInfo.GetDrives())
        {
            try
            {
                if (d.DriveType != DriveType.Fixed || !d.IsReady) continue;
                var label = string.IsNullOrWhiteSpace(d.VolumeLabel) ? "Local Disk" : d.VolumeLabel;
                list.Add(new DriveEntry(d.RootDirectory.FullName, label, d.TotalSize, d.AvailableFreeSpace));
            }
            catch { }
        }
        return list;
    }

    /// <summary>
    /// Instant listing: folder names (A→Z, not yet sized) followed by the files in this folder
    /// (real sizes, largest-first). Lets the user see and drill in immediately, before the slow
    /// recursive sizing finishes.
    /// </summary>
    public static List<UsageEntry> QuickList(string path)
    {
        var folders = new List<UsageEntry>();
        var files   = new List<UsageEntry>();

        try
        {
            foreach (var dir in Directory.GetDirectories(path))
            {
                uint at = 0; DateTime mt = default;
                try { at = (uint)File.GetAttributes(dir); mt = Directory.GetLastWriteTime(dir); } catch { }
                folders.Add(new UsageEntry { Name = Path.GetFileName(dir), FullPath = dir, IsDir = true, Sized = false, Attributes = at, Modified = mt });
            }
        }
        catch { }

        try
        {
            foreach (var f in Directory.EnumerateFiles(path))
            {
                long len = 0; uint at = 0; DateTime mt = default;
                try { var fi = new FileInfo(f); at = (uint)fi.Attributes; mt = fi.LastWriteTime; len = IsCloud(at) ? 0 : OnDiskSize(f); } catch { }
                files.Add(new UsageEntry { Name = Path.GetFileName(f), FullPath = f, IsDir = false, Bytes = len, Attributes = at, Modified = mt });
            }
        }
        catch { }

        folders.Sort((a, b) => string.Compare(a.Name, b.Name, StringComparison.OrdinalIgnoreCase));
        files.Sort((a, b) => b.Bytes.CompareTo(a.Bytes));
        return folders.Concat(files).ToList();
    }

    /// <summary>Immediate children of <paramref name="path"/>, each with its recursive size, largest first.</summary>
    public static List<UsageEntry> Children(string path, CancellationToken ct)
    {
        var entries = new List<UsageEntry>();

        // Sub-folders — recursive size, computed in parallel (this is the slow part).
        string[] dirs;
        try { dirs = Directory.GetDirectories(path); } catch { dirs = Array.Empty<string>(); }

        var dirEntries = new UsageEntry[dirs.Length];
        Parallel.For(0, dirs.Length,
            new ParallelOptions { CancellationToken = ct, MaxDegreeOfParallelism = Environment.ProcessorCount },
            i =>
            {
                var sub = dirs[i];
                long bytes = 0; int files = 0; uint at = 0; DateTime mt = default;
                try
                {
                    var attr = File.GetAttributes(sub);
                    at = (uint)attr;
                    mt = Directory.GetLastWriteTime(sub);
                    if ((attr & FileAttributes.ReparsePoint) == 0)
                        (bytes, files) = DirSize(sub, 0, ct);
                }
                catch (OperationCanceledException) { throw; }
                catch { }
                dirEntries[i] = new UsageEntry
                {
                    Name = Path.GetFileName(sub), FullPath = sub, Bytes = bytes, IsDir = true, Items = files, Attributes = at, Modified = mt,
                };
            });
        entries.AddRange(dirEntries.Where(e => e is not null));

        // Files directly in this folder.
        try
        {
            foreach (var f in Directory.EnumerateFiles(path))
            {
                ct.ThrowIfCancellationRequested();
                long len = 0; uint at = 0; DateTime mt = default;
                try { var fi = new FileInfo(f); at = (uint)fi.Attributes; mt = fi.LastWriteTime; len = IsCloud(at) ? 0 : OnDiskSize(f); } catch { }
                entries.Add(new UsageEntry { Name = Path.GetFileName(f), FullPath = f, Bytes = len, IsDir = false, Attributes = at, Modified = mt });
            }
        }
        catch (OperationCanceledException) { throw; }
        catch { }

        long total = entries.Sum(e => e.Bytes);
        foreach (var e in entries) e.Percent = total > 0 ? e.Bytes * 100.0 / total : 0;
        return entries.OrderByDescending(e => e.Bytes).ToList();
    }

    // OneDrive online-only / offline files: data lives in the cloud, ~0 on disk.
    private const uint CloudMask = 0x1000u | 0x40000u | 0x400000u;   // OFFLINE | RECALL_ON_OPEN | RECALL_ON_DATA_ACCESS
    private static bool IsCloud(uint attr) => (attr & CloudMask) != 0;

    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    private static extern uint GetCompressedFileSizeW(string path, out uint high);

    /// <summary>
    /// Actual bytes the file occupies ON DISK (allocated size). Reflects compression and, crucially,
    /// sparse files — so OneDrive "online-only" placeholders (huge logical size, ~0 on disk) count as
    /// ~0. Metadata-only, so it never triggers a cloud download. Falls back to the logical length.
    /// </summary>
    private static long OnDiskSize(string path)
    {
        uint low = GetCompressedFileSizeW(path, out uint high);
        if (low == 0xFFFFFFFF && Marshal.GetLastWin32Error() != 0)
        {
            try { return new FileInfo(path).Length; } catch { return 0; }
        }
        return ((long)high << 32) | low;
    }

    private const int MaxDepth = 40;

    private static (long bytes, int files) DirSize(string path, int depth, CancellationToken ct)
    {
        long total = 0; int count = 0;
        if (depth > MaxDepth) return (total, count);
        ct.ThrowIfCancellationRequested();
        try
        {
            foreach (var file in Directory.EnumerateFiles(path))
            {
                try { count++; if (!IsCloud((uint)File.GetAttributes(file))) total += OnDiskSize(file); } catch { }
            }
            foreach (var sub in Directory.EnumerateDirectories(path))
            {
                try
                {
                    if ((File.GetAttributes(sub) & FileAttributes.ReparsePoint) != 0) continue;
                    var (b, c) = DirSize(sub, depth + 1, ct);
                    total += b; count += c;
                }
                catch (OperationCanceledException) { throw; }
                catch { }
            }
        }
        catch (OperationCanceledException) { throw; }
        catch { }
        return (total, count);
    }
}
```
