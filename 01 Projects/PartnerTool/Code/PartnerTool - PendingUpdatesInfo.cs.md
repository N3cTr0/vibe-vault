---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PendingUpdatesInfo.cs
---

# PartnerTool\PendingUpdatesInfo.cs

```csharp
namespace PartnerTool;

public record PendingUpdate(string Title, string Kb, double SizeMb)
{
    /// <summary>
    /// Human size. WUA reports the *maximum possible* download; for UUP cumulative updates that is
    /// the entire multi-GB payload range (~90 GB!) while the real express/delta download is a small
    /// fraction — Windows doesn't expose the true size, so anything implausibly large shows as "—"
    /// (Settings hides those sizes too) instead of a scary wrong number.
    /// </summary>
    public string SizeText =>
        SizeMb <= 0     ? "" :
        SizeMb > 4096   ? "—" :
        SizeMb >= 1024  ? $"{SizeMb / 1024:F1} GB" :
        SizeMb < 1      ? $"{SizeMb * 1024:F0} KB" :
                          $"{SizeMb:F0} MB";
}

/// <summary>
/// Windows updates that are available but not yet installed, via the Windows Update Agent
/// COM API. The search talks to Windows Update so it can take a while — call it on demand
/// (a button), never on the startup path.
/// </summary>
public static class PendingUpdatesInfo
{
    public static List<PendingUpdate> Collect()
    {
        var list = new List<PendingUpdate>();
        try
        {
            var t = Type.GetTypeFromProgID("Microsoft.Update.Session");
            if (t == null) return list;
            dynamic session  = Activator.CreateInstance(t)!;
            dynamic searcher = session.CreateUpdateSearcher();
            dynamic result   = searcher.Search("IsInstalled=0 and IsHidden=0");
            dynamic updates  = result.Updates;
            int n = updates.Count;
            for (int i = 0; i < n; i++)
            {
                try
                {
                    dynamic u = updates.Item(i);
                    string title = (string)(u.Title ?? "");
                    string kb = "";
                    try { dynamic kbs = u.KBArticleIDs; if (kbs.Count > 0) kb = "KB" + kbs.Item(0); } catch { }
                    // MaxDownloadSize on the parent sums every bundled variant — and bundles often
                    // hold several mutually-exclusive applicability variants of the SAME payload (e.g.
                    // a Defender signature update lists ~7 identical ~200 MB packages). Windows only
                    // downloads one, so the realistic size is the largest single bundled package.
                    double bytes = 0;
                    try { bytes = Convert.ToInt64(u.MaxDownloadSize); } catch { }
                    try
                    {
                        dynamic bundled = u.BundledUpdates;
                        int bc = bundled.Count;
                        if (bc > 0)
                        {
                            double largest = 0;
                            for (int j = 0; j < bc; j++)
                                try { double b = Convert.ToInt64(bundled.Item(j).MaxDownloadSize); if (b > largest) largest = b; } catch { }
                            if (largest > 0) bytes = largest;
                        }
                    }
                    catch { }
                    double sizeMb = Math.Round(bytes / 1048576.0, 1);
                    if (!string.IsNullOrWhiteSpace(title)) list.Add(new PendingUpdate(title.Trim(), kb, sizeMb));
                }
                catch { }
            }
        }
        catch { }
        return list;
    }
}
```
