---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ShortcutScan.cs
---

# PartnerTool\ShortcutScan.cs

```csharp
using System.IO;
using System.Runtime.InteropServices;

namespace PartnerTool;

public class BrokenShortcut
{
    public required string LnkPath { get; init; }
    public required string Name    { get; init; }
    public required string Target  { get; init; }
}

/// <summary>
/// Finds broken shortcuts (.lnk whose target no longer exists) on the Desktop and Start Menu, and
/// can send them to the Recycle Bin (reversible, never a permanent delete). Deliberately
/// conservative to avoid false positives: only flags a shortcut whose target is an absolute path on
/// a ready FIXED drive that doesn't exist — never UNC/network or removable targets (which may just
/// be temporarily offline), and never shortcuts with an empty target (URLs, special items).
/// </summary>
public static class ShortcutScan
{
    private static string[] Roots() => new[]
    {
        Environment.GetFolderPath(Environment.SpecialFolder.DesktopDirectory),
        Environment.GetFolderPath(Environment.SpecialFolder.CommonDesktopDirectory),
        Environment.GetFolderPath(Environment.SpecialFolder.Programs),
        Environment.GetFolderPath(Environment.SpecialFolder.CommonPrograms),
    }.Where(p => p.Length > 0).Distinct(StringComparer.OrdinalIgnoreCase).ToArray();

    public static List<BrokenShortcut> Scan()
    {
        var list = new List<BrokenShortcut>();
        dynamic? shell = null;
        try
        {
            var t = Type.GetTypeFromProgID("WScript.Shell");
            if (t == null) return list;
            shell = Activator.CreateInstance(t);

            foreach (var root in Roots())
            {
                IEnumerable<string> lnks;
                try { lnks = Directory.EnumerateFiles(root, "*.lnk", SearchOption.AllDirectories); }
                catch { continue; }

                foreach (var lnk in lnks)
                {
                    try
                    {
                        dynamic sc = shell!.CreateShortcut(lnk);
                        string target = (string)(sc.TargetPath ?? "");
                        if (IsBrokenTarget(target))
                            list.Add(new BrokenShortcut
                            {
                                LnkPath = lnk,
                                Name    = Path.GetFileNameWithoutExtension(lnk),
                                Target  = target,
                            });
                    }
                    catch { /* unreadable shortcut — skip */ }
                }
            }
        }
        catch { }
        finally { if (shell != null) { try { Marshal.FinalReleaseComObject(shell); } catch { } } }
        return list;
    }

    private static bool IsBrokenTarget(string target)
    {
        if (string.IsNullOrWhiteSpace(target)) return false;   // URL / special item
        if (target.StartsWith(@"\\")) return false;            // UNC — may be offline
        if (!Path.IsPathFullyQualified(target)) return false;
        try
        {
            var root = Path.GetPathRoot(target);
            if (string.IsNullOrEmpty(root)) return false;
            var drive = new DriveInfo(root);
            if (drive.DriveType != DriveType.Fixed || !drive.IsReady) return false;   // removable/offline
        }
        catch { return false; }
        return !File.Exists(target) && !Directory.Exists(target);
    }

    /// <summary>Send the given .lnk files to the Recycle Bin (reversible). Returns the count moved.</summary>
    public static int Recycle(IEnumerable<string> lnkPaths, Action<string>? onLog = null)
    {
        int n = 0;
        foreach (var p in lnkPaths)
        {
            try
            {
                if (RecycleFile(p)) { n++; onLog?.Invoke($"  Recycled: {p}"); }
                else onLog?.Invoke($"  Could not recycle: {p}");
            }
            catch (Exception ex) { onLog?.Invoke($"  Skipped {p}: {ex.Message}"); }
        }
        return n;
    }

    // ── SHFileOperation → Recycle Bin (FOF_ALLOWUNDO), no UI ──
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    private struct SHFILEOPSTRUCTW
    {
        public IntPtr hwnd;
        public uint   wFunc;
        public IntPtr pFrom;
        public IntPtr pTo;
        public ushort fFlags;
        [MarshalAs(UnmanagedType.Bool)] public bool fAnyOperationsAborted;
        public IntPtr hNameMappings;
        public IntPtr lpszProgressTitle;
    }

    [DllImport("shell32.dll", CharSet = CharSet.Unicode)]
    private static extern int SHFileOperationW(ref SHFILEOPSTRUCTW lpFileOp);

    private const uint   FO_DELETE          = 0x0003;
    private const ushort FOF_SILENT         = 0x0004;
    private const ushort FOF_NOCONFIRMATION = 0x0010;
    private const ushort FOF_ALLOWUNDO      = 0x0040;
    private const ushort FOF_NOERRORUI      = 0x0400;

    private static bool RecycleFile(string path)
    {
        if (!File.Exists(path)) return false;
        // Double-null-terminated file list, allocated explicitly so embedded nulls survive marshalling.
        IntPtr pFrom = Marshal.StringToHGlobalUni(path + "\0\0");
        try
        {
            var op = new SHFILEOPSTRUCTW
            {
                wFunc  = FO_DELETE,
                pFrom  = pFrom,
                fFlags = (ushort)(FOF_ALLOWUNDO | FOF_NOCONFIRMATION | FOF_SILENT | FOF_NOERRORUI),
            };
            return SHFileOperationW(ref op) == 0;
        }
        finally { Marshal.FreeHGlobal(pFrom); }
    }
}
```
