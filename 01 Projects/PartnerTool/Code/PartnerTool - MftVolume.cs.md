---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\MftVolume.cs
---

# PartnerTool\MftVolume.cs

```csharp
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;
using Microsoft.Win32.SafeHandles;

namespace PartnerTool;

/// <summary>
/// WizTree-style fast disk scan: reads the raw NTFS Master File Table ($MFT) in one sequential pass
/// (read-only — the app runs elevated, which is the one requirement) to enumerate every file and
/// folder with its size, then builds the whole directory tree in memory so drill-downs are instant.
///
/// NTFS fixed drives only. <see cref="TryScan"/> returns null on anything else, or on any failure /
/// parse error, and the caller falls back to the recursive folder walk in <see cref="DiskUsageInfo"/>.
/// Opening \\.\X: with GENERIC_READ is read-only — no writes, nothing executed — so there is no data
/// or privilege-escalation risk beyond the admin rights the tool already has.
/// </summary>
public sealed class MftVolume
{
    // ── Win32 ────────────────────────────────────────────────────────────
    private const uint GENERIC_READ = 0x80000000;
    private const uint FILE_SHARE_RW = 0x00000003;
    private const uint OPEN_EXISTING = 3;
    private const uint FSCTL_GET_NTFS_VOLUME_DATA = 0x00090064;

    [StructLayout(LayoutKind.Sequential)]
    private struct NTFS_VOLUME_DATA_BUFFER
    {
        public long VolumeSerialNumber, NumberSectors, TotalClusters, FreeClusters, TotalReserved;
        public uint BytesPerSector, BytesPerCluster, BytesPerFileRecordSegment, ClustersPerFileRecordSegment;
        public long MftValidDataLength, MftStartLcn, Mft2StartLcn, MftZoneStart, MftZoneEnd;
    }

    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    private static extern SafeFileHandle CreateFileW(string name, uint access, uint share, IntPtr sec,
        uint disposition, uint flags, IntPtr template);

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool DeviceIoControl(SafeFileHandle h, uint code, IntPtr inBuf, uint inSize,
        out NTFS_VOLUME_DATA_BUFFER outBuf, uint outSize, out uint returned, IntPtr overlapped);

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool SetFilePointerEx(SafeFileHandle h, long distance, out long newPos, uint method);

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool ReadFile(SafeFileHandle h, byte[] buffer, uint toRead, out uint read, IntPtr overlapped);

    // ── Parsed volume ────────────────────────────────────────────────────
    private readonly int      _count;
    private readonly string?[] _name;
    private readonly int[]     _parent;   // parent record #, or -1
    private readonly long[]    _size;     // own $DATA bytes; 0 for folders
    private readonly uint[]    _attr;     // $STANDARD_INFORMATION DOS attributes (has Hidden/System)
    private readonly long[]    _mtime;    // $STANDARD_INFORMATION last-modified FILETIME
    private readonly bool[]    _isDir;
    private readonly bool[]    _inUse;
    private readonly List<int>?[] _children;
    private readonly long[]    _total;    // recursive size
    private readonly int[]     _files;    // recursive file count
    private readonly bool[]    _done;

    public string Root { get; }
    private const int RootRec = 5;        // NTFS root directory is always MFT record 5

    private MftVolume(string root, int count, string?[] name, int[] parent, long[] size, uint[] attr, long[] mtime, bool[] isDir, bool[] inUse)
    {
        Root = root; _count = count; _name = name; _parent = parent; _size = size; _attr = attr; _mtime = mtime; _isDir = isDir; _inUse = inUse;
        _children = new List<int>?[count];
        _total = new long[count];
        _files = new int[count];
        _done  = new bool[count];

        for (int i = 0; i < count; i++)
        {
            if (i == RootRec || !inUse[i] || name[i] == null) continue;
            int p = parent[i];
            if (p < 0 || p >= count || p == i) continue;
            (_children[p] ??= new List<int>()).Add(i);
        }
        Total(RootRec, 0);
    }

    private (long bytes, int files) Total(int i, int depth)
    {
        if (_done[i]) return (_total[i], _files[i]);
        _done[i] = true;
        long bytes = _isDir[i] ? 0 : _size[i];
        int  files = _isDir[i] ? 0 : 1;
        var kids = _children[i];
        if (kids != null && depth < 1024)
            foreach (var c in kids) { var (cb, cf) = Total(c, depth + 1); bytes += cb; files += cf; }
        _total[i] = bytes; _files[i] = files;
        return (bytes, files);
    }

    /// <summary>Immediate children of <paramref name="path"/> from the in-memory tree — instant.</summary>
    public List<UsageEntry> Children(string path)
    {
        var list = new List<UsageEntry>();
        int dir = Resolve(path);
        if (dir < 0) return list;
        var kids = _children[dir];
        if (kids != null)
            foreach (var c in kids)
            {
                long bytes = _isDir[c] ? _total[c] : _size[c];
                list.Add(new UsageEntry
                {
                    Name       = _name[c]!,
                    FullPath   = Path.Combine(path, _name[c]!),
                    Bytes      = bytes,
                    IsDir      = _isDir[c],
                    Items      = _isDir[c] ? _files[c] : 0,
                    Attributes = _attr[c],
                    Modified   = FromFileTime(_mtime[c]),
                });
            }
        long total = list.Sum(e => e.Bytes);
        foreach (var e in list) e.Percent = total > 0 ? e.Bytes * 100.0 / total : 0;
        return list.OrderByDescending(e => e.Bytes).ToList();
    }

    private int Resolve(string path)
    {
        try
        {
            var full = Path.GetFullPath(path);
            var root = Path.GetPathRoot(full) ?? "";
            var rest = full[root.Length..].Trim('\\');
            int cur = RootRec;
            if (rest.Length == 0) return cur;
            foreach (var part in rest.Split('\\', StringSplitOptions.RemoveEmptyEntries))
            {
                var kids = _children[cur];
                if (kids == null) return -1;
                int next = -1;
                foreach (var c in kids)
                    if (_isDir[c] && string.Equals(_name[c], part, StringComparison.OrdinalIgnoreCase)) { next = c; break; }
                if (next < 0) return -1;
                cur = next;
            }
            return cur;
        }
        catch { return -1; }
    }

    // ── Scan ─────────────────────────────────────────────────────────────
    public static MftVolume? TryScan(string root, CancellationToken ct)
    {
        try
        {
            if (root.Length < 2 || root[1] != ':') return null;
            var letter = root[..2];   // "C:"
            if (!string.Equals(new DriveInfo(letter).DriveFormat, "NTFS", StringComparison.OrdinalIgnoreCase)) return null;

            using var h = CreateFileW($@"\\.\{letter}", GENERIC_READ, FILE_SHARE_RW, IntPtr.Zero, OPEN_EXISTING, 0, IntPtr.Zero);
            if (h.IsInvalid) return null;

            if (!DeviceIoControl(h, FSCTL_GET_NTFS_VOLUME_DATA, IntPtr.Zero, 0, out var vd,
                    (uint)Marshal.SizeOf<NTFS_VOLUME_DATA_BUFFER>(), out _, IntPtr.Zero))
                return null;

            int recSize = (int)vd.BytesPerFileRecordSegment;
            int cluster = (int)vd.BytesPerCluster;
            if (recSize < 512 || cluster < 512) return null;

            long recordCountL = vd.MftValidDataLength / recSize;
            if (recordCountL <= 0 || recordCountL > 60_000_000) return null;   // sanity
            int recordCount = (int)recordCountL;

            // Record 0 ($MFT) → its own data runs (the MFT can be fragmented).
            var rec0 = new byte[Align(recSize, cluster)];
            if (!ReadAt(h, vd.MftStartLcn * cluster, rec0, rec0.Length)) return null;
            var runs = MftDataRuns(rec0, recSize, cluster);
            if (runs is null || runs.Count == 0) return null;

            var name   = new string?[recordCount];
            var parent = new int[recordCount];
            var size   = new long[recordCount];
            var attr   = new uint[recordCount];
            var mtime  = new long[recordCount];
            var cloud  = new bool[recordCount];
            var isDir  = new bool[recordCount];
            var inUse  = new bool[recordCount];

            int index = 0;
            int blockClusters = Math.Max(1, (1 << 20) / cluster);        // ~1 MB blocks
            var block = new byte[blockClusters * cluster];

            foreach (var (lcn, lenClusters) in runs)
            {
                if (index >= recordCount) break;
                long pos = lcn * cluster;
                long remaining = lenClusters * cluster;
                while (remaining > 0 && index < recordCount)
                {
                    ct.ThrowIfCancellationRequested();
                    int want = (int)Math.Min(block.Length, remaining);
                    want -= want % cluster;
                    if (want <= 0) break;
                    if (!ReadAt(h, pos, block, want)) return null;
                    pos += want; remaining -= want;

                    for (int off = 0; off + recSize <= want && index < recordCount; off += recSize, index++)
                    {
                        try { ParseRecord(block, off, recSize, cluster, index, name, parent, size, attr, mtime, cloud, isDir, inUse); }
                        catch { /* skip a malformed record */ }
                    }
                }
            }

            for (int i = 0; i < recordCount; i++)
            {
                // Zero OneDrive cloud placeholders — data lives in the cloud, not on disk. Use ONLY the
                // precise IO_REPARSE_TAG_CLOUD reparse tag; the $STANDARD_INFORMATION attribute field is
                // not a clean GetFileAttributes and falsely flagged normal Windows/Program Files files.
                if (cloud[i]) size[i] = 0;
                // $BadClus is a phantom: NTFS declares its $Bad stream's AllocatedSize as the whole
                // volume (bad-sector map) but it occupies ~0 real bytes. WizTree hides it; we zero it.
                else if (name[i] == "$BadClus") size[i] = 0;
            }

            return new MftVolume(letter + "\\", recordCount, name, parent, size, attr, mtime, isDir, inUse);
        }
        catch { return null; }
    }

    private static int Align(int v, int a) => ((v + a - 1) / a) * a;

    private static DateTime FromFileTime(long ft)
    {
        try { return ft > 0 ? DateTime.FromFileTime(ft) : default; } catch { return default; }
    }

    private static bool ReadAt(SafeFileHandle h, long offset, byte[] buf, int count)
    {
        if (!SetFilePointerEx(h, offset, out _, 0)) return false;      // FILE_BEGIN
        return ReadFile(h, buf, (uint)count, out uint read, IntPtr.Zero) && read == count;
    }

    private static void ParseRecord(byte[] b, int off, int recSize, int cluster, int index,
        string?[] name, int[] parent, long[] size, uint[] attr, long[] mtime, bool[] cloud, bool[] isDir, bool[] inUse)
    {
        parent[index] = -1;
        if (b[off] != (byte)'F' || b[off + 1] != (byte)'I' || b[off + 2] != (byte)'L' || b[off + 3] != (byte)'E') return;
        if (!ApplyFixup(b, off, recSize)) return;

        ushort flags = U16(b, off + 0x16);
        bool used = (flags & 0x01) != 0;
        inUse[index] = used;
        if (!used) return;
        isDir[index] = (flags & 0x02) != 0;

        // Overflow ($ATTRIBUTE_LIST) records carry a non-zero base reference at +0x20; a $DATA size
        // found here belongs to that BASE file (e.g. a huge fragmented VM disk), not this record.
        int target = index;
        long baseRef = (long)(U64(b, off + 0x20) & 0xFFFFFFFFFFFF);
        if (baseRef != 0 && baseRef < size.Length) target = (int)baseRef;

        int a = off + U16(b, off + 0x14);   // first attribute
        int end = off + recSize;
        int bestNs = -1;
        while (a + 8 <= end)
        {
            uint type = U32(b, a);
            if (type == 0xFFFFFFFF) break;
            uint len = U32(b, a + 4);
            if (len < 8 || a + (int)len > end) break;
            byte nonRes = b[a + 8];

            if (type == 0x10 && nonRes == 0)                    // $STANDARD_INFORMATION → attributes + modified time
            {
                int v = a + U16(b, a + 0x14);
                if (v + 0x10 <= end) mtime[index] = I64(b, v + 0x08);   // last-modified FILETIME
                if (v + 0x24 <= end) attr[index]  = U32(b, v + 0x20);   // DOS attributes
            }
            else if (type == 0x30 && nonRes == 0)               // $FILE_NAME → name + parent
            {
                int v = a + U16(b, a + 0x14);
                if (v + 0x42 <= end)
                {
                    int nl = b[v + 0x40];
                    int ns = b[v + 0x41];
                    if (ns != 2 && ns > bestNs && v + 0x42 + nl * 2 <= end)  // prefer Win32/Win32&DOS over the DOS short name
                    {
                        bestNs = ns;
                        parent[index] = (int)(U64(b, v) & 0xFFFFFFFFFFFF);
                        name[index]   = System.Text.Encoding.Unicode.GetString(b, v + 0x42, nl * 2);
                    }
                }
            }
            else if (type == 0x80)                              // $DATA stream(s) → sum size ON DISK
            {
                // Sum EVERY data stream, not just the unnamed one: WOF/CompactOS-compressed Windows
                // files keep their real data in a "WofCompressedData" named stream (the unnamed one
                // is ~0), and alternate data streams take space too — WizTree counts all of them.
                if (nonRes == 0) size[target] += U32(b, a + 0x10);                         // resident stream (tiny)
                else if (a + 0x30 <= end && I64(b, a + 0x10) == 0)                         // only the StartingVCN=0 header
                {
                    if ((U16(b, a + 0x0C) & 0x8000) != 0)                                  // SPARSE stream
                    {
                        // AllocatedSize is unreliable for sparse system files ($BadClus reserves the
                        // whole volume, $UsnJrnl the journal max), so total the real (non-sparse) runs.
                        long clusters = 0;
                        foreach (var (_, runLen) in DecodeRuns(b, a + U16(b, a + 0x20), a + (int)len)) clusters += runLen;
                        size[target] += clusters * cluster;
                    }
                    else
                        size[target] += I64(b, a + 0x28);                                  // AllocatedSize (handles fragmented + WOF)
                }
            }
            else if (type == 0xC0 && nonRes == 0)               // $REPARSE_POINT — flag OneDrive cloud placeholders
            {
                // A cloud placeholder's AllocatedSize equals its logical size, so it would wrongly
                // count as local space. Detect the IO_REPARSE_TAG_CLOUD* family (0x9000XX1A) and
                // zero it later — the file's data lives in the cloud, not on disk.
                int v = a + U16(b, a + 0x14);
                if (v + 4 <= end && (U32(b, v) & 0xFFFF0FFF) == 0x9000001A) cloud[target] = true;
            }

            a += (int)len;
        }
    }

    // Update-Sequence-Array fixup: the last 2 bytes of every 512-byte sector are the check value;
    // the real bytes live in the USA and must be written back before the record is trusted.
    private static bool ApplyFixup(byte[] b, int off, int recSize)
    {
        ushort usaOff = U16(b, off + 0x04);
        ushort usaCnt = U16(b, off + 0x06);
        if (usaCnt < 2) return false;
        int sectors = usaCnt - 1;
        int sectorSize = recSize / sectors;
        if (sectorSize < 512 || off + usaOff + usaCnt * 2 > off + recSize) return false;
        for (int i = 0; i < sectors; i++)
        {
            int tail = off + (i + 1) * sectorSize - 2;
            int src  = off + usaOff + 2 + i * 2;
            if (tail + 2 > off + recSize) return false;
            b[tail]     = b[src];
            b[tail + 1] = b[src + 1];
        }
        return true;
    }

    private static List<(long lcn, long len)>? MftDataRuns(byte[] b, int recSize, int cluster)
    {
        if (b[0] != (byte)'F' || b[1] != (byte)'I' || b[2] != (byte)'L' || b[3] != (byte)'E') return null;
        if (!ApplyFixup(b, 0, recSize)) return null;
        int a = U16(b, 0x14);
        while (a + 8 <= recSize)
        {
            uint type = U32(b, a);
            if (type == 0xFFFFFFFF) break;
            uint len = U32(b, a + 4);
            if (len < 8 || a + (int)len > recSize) break;
            if (type == 0x80 && b[a + 9] == 0 && b[a + 8] == 1)   // unnamed, non-resident $DATA
                return DecodeRuns(b, a + U16(b, a + 0x20), a + (int)len);
            a += (int)len;
        }
        return null;
    }

    private static List<(long lcn, long len)> DecodeRuns(byte[] b, int p, int end)
    {
        var runs = new List<(long, long)>();
        long lcn = 0;
        while (p < end)
        {
            byte header = b[p++];
            if (header == 0) break;
            int lenBytes = header & 0x0F;
            int offBytes = (header >> 4) & 0x0F;
            if (lenBytes == 0 || p + lenBytes + offBytes > end) break;
            long runLen = VarUnsigned(b, p, lenBytes); p += lenBytes;
            long runOff = VarSigned(b, p, offBytes);   p += offBytes;
            lcn += runOff;
            if (offBytes > 0 && runLen > 0) runs.Add((lcn, runLen));   // skip sparse runs
        }
        return runs;
    }

    // ── little-endian helpers ────────────────────────────────────────────
    private static ushort U16(byte[] b, int o) => (ushort)(b[o] | (b[o + 1] << 8));
    private static uint   U32(byte[] b, int o) => (uint)b[o] | ((uint)b[o + 1] << 8) | ((uint)b[o + 2] << 16) | ((uint)b[o + 3] << 24);
    private static ulong  U64(byte[] b, int o) { ulong v = 0; for (int i = 7; i >= 0; i--) v = (v << 8) | (uint)b[o + i]; return v; }
    private static long   I64(byte[] b, int o) { long v = 0; for (int i = 7; i >= 0; i--) v = (v << 8) | (uint)b[o + i]; return v; }

    private static long VarUnsigned(byte[] b, int o, int n)
    {
        long v = 0; for (int i = n - 1; i >= 0; i--) v = (v << 8) | (uint)b[o + i]; return v;
    }

    private static long VarSigned(byte[] b, int o, int n)
    {
        long v = 0; for (int i = n - 1; i >= 0; i--) v = (v << 8) | (uint)b[o + i];
        if (n > 0 && (b[o + n - 1] & 0x80) != 0) v |= -1L << (n * 8);   // sign-extend
        return v;
    }
}
```
