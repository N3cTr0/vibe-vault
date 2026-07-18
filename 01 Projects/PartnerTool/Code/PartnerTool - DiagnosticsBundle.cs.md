---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DiagnosticsBundle.cs
---

# PartnerTool\DiagnosticsBundle.cs

```csharp
using System.IO;
using System.IO.Compression;

namespace PartnerTool;

/// <summary>
/// "Collect diagnostics" — bundles the system report, summary, and the tool's own logs into
/// a single zip a tech can attach to a ticket.
/// </summary>
public static class DiagnosticsBundle
{
    public static async Task CreateAsync(string destinationZip, SystemSnapshot snap)
    {
        // Gather everything the tool collects (snapshot + software / startup / adapters / Wi-Fi /
        // hardening), then build the full report off the UI thread.
        var data = await FullReport.GatherAsync(snap);
        var html = ReportBuilder.BuildHtml(data);
        var text = ReportBuilder.BuildText(data);

        await Task.Run(() =>
        {
            if (File.Exists(destinationZip)) File.Delete(destinationZip);
            using var zip = ZipFile.Open(destinationZip, ZipArchiveMode.Create);

            AddText(zip, "SystemReport.html", html);
            AddText(zip, "SystemSummary.txt", text);

            // Tool logs (repair/quick-fix/error logs)
            TryAddFolder(zip, @"C:\PCI\Logs", "Logs");

            // Config context, if present next to the exe
            TryAddFile(zip, Path.Combine(AppContext.BaseDirectory, "settings.json"), "settings.json");
            TryAddFile(zip, Path.Combine(AppContext.BaseDirectory, "history.json"),  "history.json");
        });
    }

    private static void AddText(ZipArchive zip, string entryName, string content)
    {
        try
        {
            var e = zip.CreateEntry(entryName);
            using var w = new StreamWriter(e.Open());
            w.Write(content);
        }
        catch { }
    }

    private static void TryAddFile(ZipArchive zip, string path, string entryName)
    {
        try { if (File.Exists(path)) zip.CreateEntryFromFile(path, entryName); }
        catch { }
    }

    private static void TryAddFolder(ZipArchive zip, string dir, string entryFolder)
    {
        try
        {
            if (!Directory.Exists(dir)) return;
            foreach (var f in Directory.GetFiles(dir, "*", SearchOption.AllDirectories))
            {
                try
                {
                    // Read with ReadWrite share so the tool's OWN currently-open logs (activity /
                    // errors, appended to as we run) don't fail with a sharing violation and get
                    // silently dropped — CreateEntryFromFile opens with only FileShare.Read.
                    var rel = Path.GetRelativePath(dir, f).Replace('\\', '/');
                    var entry = zip.CreateEntry($"{entryFolder}/{rel}");
                    using var src = new FileStream(f, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
                    using var dst = entry.Open();
                    src.CopyTo(dst);
                }
                catch { }
            }
        }
        catch { }
    }
}
```
