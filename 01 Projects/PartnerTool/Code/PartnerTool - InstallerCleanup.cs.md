---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\InstallerCleanup.cs
---

# PartnerTool\InstallerCleanup.cs

```csharp
using System.IO;

namespace PartnerTool;

public class OrphanFile
{
    public string Path        = "";
    public string Name        = "";
    public long   Bytes;
    public string Description = "";      // what the package is for (read from MSI/MSP metadata)
    public double Mb => Bytes / 1048576.0;
}

public class InstallerScanResult
{
    public int              ReferencedCount;
    public List<OrphanFile> Orphans = new();
    public long             OrphanBytes;
    public bool             KeepSetValid;
    public string?          Error;

    public double OrphanGb => OrphanBytes / 1073741824.0;

    /// <summary>Orphans grouped by what they're for, largest first — for the confirm dialog.</summary>
    public IEnumerable<(string Product, int Count, long Bytes)> TopProducts(int top) =>
        Orphans.GroupBy(o => o.Description.Length > 0 ? o.Description : "(unidentified)")
               .Select(g => (Product: g.Key, Count: g.Count(), Bytes: g.Sum(o => o.Bytes)))
               .OrderByDescending(x => x.Bytes)
               .Take(top);
}

/// <summary>
/// Cleans orphaned .msi/.msp files out of C:\Windows\Installer — the classic Adobe Acrobat/Reader
/// bloat where the updater caches monthly patches and never removes the superseded ones.
///
/// SAFETY: the Windows Installer cache is NOT junk — deleting a package still referenced by an
/// installed product breaks its repair/uninstall/patching. So we ask Windows Installer itself
/// (WindowsInstaller.Installer COM) for the authoritative set of cached packages every installed
/// product and patch still needs, and remove ONLY files on disk that aren't in that set. If that
/// set can't be read (or comes back empty), we abort. Each orphan's product metadata is read
/// read-only (no extraction/execution) so the log shows what every file was for.
/// </summary>
public static class InstallerCleanup
{
    private const string InstallerDir = @"C:\Windows\Installer";

    // Windows Installer open modes
    private const int ReadOnly  = 0;
    private const int PatchFile = 32;   // msiOpenDatabaseModePatchFile
    // SummaryInformation property IDs
    private const int PidTitle = 2, PidSubject = 3, PidAuthor = 4, PidComments = 6;

    public static InstallerScanResult Scan()
    {
        var r = new InstallerScanResult();
        var keep = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        dynamic? msi = null;

        try
        {
            var t = Type.GetTypeFromProgID("WindowsInstaller.Installer");
            if (t == null) { r.Error = "Windows Installer automation is unavailable."; return r; }
            msi = Activator.CreateInstance(t)!;

            dynamic products = msi.Products;                 // all installed product codes
            int pc = products.Count;
            for (int i = 0; i < pc; i++)
            {
                string product;
                try { product = (string)products.Item(i); } catch { continue; }

                try { AddLocalPackage(keep, (string)msi.ProductInfo(product, "LocalPackage")); } catch { }

                try
                {
                    dynamic patches = msi.Patches(product);
                    int qc = patches.Count;
                    for (int j = 0; j < qc; j++)
                    {
                        try
                        {
                            string patch = (string)patches.Item(j);
                            AddLocalPackage(keep, (string)msi.PatchInfo(patch, "LocalPackage"));
                        }
                        catch { }
                    }
                }
                catch { }
            }
        }
        catch (Exception ex)
        {
            r.Error = "Couldn't read the Windows Installer database: " + ex.Message;
            return r;
        }

        r.ReferencedCount = keep.Count;
        r.KeepSetValid = keep.Count > 0;
        if (!r.KeepSetValid)
        {
            r.Error = "The installer database returned no referenced packages — aborting so nothing referenced can be deleted.";
            return r;
        }

        try
        {
            var di = new DirectoryInfo(InstallerDir);
            if (di.Exists)
                foreach (var f in di.EnumerateFiles())   // top level only — never the {GUID} subfolders
                {
                    bool msiFile = f.Extension.Equals(".msi", StringComparison.OrdinalIgnoreCase);
                    bool mspFile = f.Extension.Equals(".msp", StringComparison.OrdinalIgnoreCase);
                    if (!msiFile && !mspFile) continue;
                    if (keep.Contains(f.FullName)) continue;

                    r.Orphans.Add(new OrphanFile
                    {
                        Path        = f.FullName,
                        Name        = f.Name,
                        Bytes       = f.Length,
                        Description = Describe(msi, f.FullName, mspFile),
                    });
                    r.OrphanBytes += f.Length;
                }
        }
        catch (Exception ex) { r.Error = $"Couldn't enumerate {InstallerDir}: {ex.Message}"; }

        return r;
    }

    public static (int deleted, long freed, int failed) DeleteOrphans(IEnumerable<OrphanFile> orphans, Action<string>? onLog)
    {
        int deleted = 0, failed = 0; long freed = 0;
        foreach (var o in orphans)
        {
            try
            {
                var fi = new FileInfo(o.Path);
                if (fi.IsReadOnly) fi.IsReadOnly = false;
                fi.Delete();
                deleted++; freed += o.Bytes;
                onLog?.Invoke(Line("deleted", o));
            }
            catch (Exception ex) { failed++; onLog?.Invoke($"skipped {o.Name} — {ex.Message}"); }
        }
        return (deleted, freed, failed);
    }

    /// <summary>One log line: "name (size) — description".</summary>
    public static string Line(string verb, OrphanFile o)
        => $"{verb} {o.Name}  ({o.Mb:F0} MB)" + (o.Description.Length > 0 ? $"  —  {o.Description}" : "");

    private static void AddLocalPackage(HashSet<string> keep, string? path)
    {
        if (!string.IsNullOrWhiteSpace(path))
            try { keep.Add(Path.GetFullPath(path)); } catch { keep.Add(path); }
    }

    // ── Read what a cached package is for (read-only; opens the DB, never runs it) ──
    private static string Describe(dynamic msi, string path, bool isPatch)
    {
        try
        {
            if (isPatch)
            {
                // Patches have no Property table — use the summary stream (Subject = product).
                dynamic db = msi.OpenDatabase(path, PatchFile);
                dynamic si = db.SummaryInformation(0);
                var d = FirstNonEmpty(Si(si, PidSubject), Si(si, PidComments), Si(si, PidTitle));
                return d.Length > 0 ? d + "  (patch)" : "patch";
            }
            else
            {
                dynamic db = msi.OpenDatabase(path, ReadOnly);
                string product = Prop(db, "ProductName");
                string maker   = Prop(db, "Manufacturer");
                string ver     = Prop(db, "ProductVersion");
                var parts = new List<string>();
                if (maker.Length   > 0) parts.Add(maker);
                if (product.Length > 0) parts.Add(product);
                if (ver.Length     > 0) parts.Add("v" + ver);
                if (parts.Count > 0) return string.Join(" ", parts);

                dynamic si = db.SummaryInformation(0);
                return FirstNonEmpty(Si(si, PidSubject), Si(si, PidComments));
            }
        }
        catch { return ""; }   // corrupt / 0-byte / unreadable — leave blank
    }

    private static string Prop(dynamic db, string property)
    {
        try
        {
            dynamic view = db.OpenView($"SELECT `Value` FROM `Property` WHERE `Property`='{property}'");
            view.Execute();
            dynamic rec = view.Fetch();
            return rec != null ? ((string)rec.StringData(1)).Trim() : "";
        }
        catch { return ""; }
    }

    private static string Si(dynamic si, int pid)
    {
        try { return (si.Property(pid) as string ?? "").Replace("\r", " ").Replace("\n", " ").Trim(); }
        catch { return ""; }
    }

    private static string FirstNonEmpty(params string[] xs)
        => xs.FirstOrDefault(x => !string.IsNullOrWhiteSpace(x))?.Trim() ?? "";

    /// <summary>
    /// Adobe's own prevention fix: PatchCleanFlag=1 makes Acrobat/Reader's updater purge old cached
    /// patches on the next update instead of hoarding them. Harmless policy value under FeatureLockdown.
    /// </summary>
    public static (bool ok, string message) ApplyAdobePatchCleanFix()
    {
        var targets = new[]
        {
            @"SOFTWARE\WOW6432Node\Policies\Adobe\Acrobat Reader\DC\FeatureLockdown",
            @"SOFTWARE\Policies\Adobe\Adobe Acrobat\DC\FeatureLockdown",
        };

        int set = 0; var errs = new List<string>();
        foreach (var key in targets)
        {
            try
            {
                using var k = Microsoft.Win32.Registry.LocalMachine.CreateSubKey(key, writable: true);
                k.SetValue("PatchCleanFlag", 1, Microsoft.Win32.RegistryValueKind.DWord);
                set++;
            }
            catch (Exception ex) { errs.Add(ex.Message); }
        }

        return set > 0
            ? (true, $"Set PatchCleanFlag=1 in {set} Adobe policy key(s) — Acrobat/Reader will purge old cached patches on its next update.")
            : (false, "Couldn't set the Adobe policy value. " + string.Join("; ", errs));
    }
}
```
