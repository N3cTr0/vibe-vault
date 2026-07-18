---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\LogRetention.cs
---

# PartnerTool\LogRetention.cs

```csharp
using System.IO;

namespace PartnerTool;

/// <summary>
/// Keeps C:\PCI\Logs from growing forever. The folder is ACL-hardened (only elevated contexts can
/// write or delete), so techs can't tidy it by hand — the elevated app prunes it at startup
/// instead, deleting log files whose last write is older than the configured retention
/// (Settings ▸ Log retention, default 30 days). The always-appended activity log's LastWriteTime
/// refreshes every session, so it survives as long as the tool is actually in use; a stale
/// errors.log from long-fixed crashes ages out naturally.
/// </summary>
public static class LogRetention
{
    private const string Dir = @"C:\PCI\Logs";

    public static void Prune()
    {
        try
        {
            int days   = Math.Clamp(SettingsStore.Current.LogRetentionDays, 1, 365);
            var cutoff = DateTime.Now.AddDays(-days);
            var dir    = new DirectoryInfo(Dir);
            if (!dir.Exists) return;

            int count = 0; long bytes = 0;
            foreach (var f in dir.GetFiles())
            {
                try
                {
                    if (f.LastWriteTime >= cutoff) continue;
                    long len = f.Length;
                    f.Delete();
                    count += 1; bytes += len;
                }
                catch { /* locked / in use — the next launch gets it */ }
            }

            if (count > 0)
                ActivityLog.Action("Logs",
                    $"Pruned {count} log file(s) older than {days} day(s) ({bytes / 1048576.0:F1} MB)");
        }
        catch { /* housekeeping must never break startup */ }
    }
}
```
