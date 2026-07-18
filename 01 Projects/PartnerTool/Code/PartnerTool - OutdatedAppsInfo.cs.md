---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\OutdatedAppsInfo.cs
---

# PartnerTool\OutdatedAppsInfo.cs

```csharp
namespace PartnerTool;

public record OutdatedApp(string Name, string Id, string Current, string Available);

/// <summary>
/// Third-party apps with updates available, parsed from <c>winget upgrade</c>. Lets a tech
/// see (and, via the Updates tab, action) outdated software in one place.
/// </summary>
public static class OutdatedAppsInfo
{
    public static async Task<List<OutdatedApp>> CollectAsync()
    {
        var list = new List<OutdatedApp>();
        try
        {
            var winget = WingetLocator.Path();
            if (winget.Length == 0) return list;   // not usable by this account — nothing to report

            // No --include-unknown: entries whose current version winget can't read ("Unknown → x.y")
            // are noise the techs don't act on, so leave them out of the list.
            var text = await ProcessRunner.RunCaptureAsync(winget,
                "upgrade --accept-source-agreements", 60000);
            var lines = text.Replace("\r", "").Split('\n');

            // Find the header row to locate column start positions (winget is column-aligned).
            int headerIdx = Array.FindIndex(lines, l =>
                l.Contains("Name") && l.Contains("Id") && l.Contains("Version") && l.Contains("Available"));
            if (headerIdx < 0) return list;

            var header = lines[headerIdx];
            int cName = header.IndexOf("Name", StringComparison.Ordinal);
            int cId   = header.IndexOf("Id", StringComparison.Ordinal);
            int cVer  = header.IndexOf("Version", StringComparison.Ordinal);
            int cAvail= header.IndexOf("Available", StringComparison.Ordinal);
            int cSrc  = header.IndexOf("Source", StringComparison.Ordinal);
            if (cName < 0 || cId <= cName || cVer <= cId || cAvail <= cVer) return list;

            for (int i = headerIdx + 1; i < lines.Length; i++)
            {
                var line = lines[i];
                if (string.IsNullOrWhiteSpace(line)) continue;
                if (line.StartsWith("-") || line.TrimStart().StartsWith("---")) continue;
                // Stop at winget's summary footers — "N upgrades available." and the "M package(s)
                // have version numbers that cannot be determined. Use --include-unknown …" line,
                // which otherwise gets column-sliced into a bogus "app" row.
                if (line.Contains("upgrades available") || line.Contains("package(s)")) break;
                if (line.Length < cAvail) continue;

                string Slice(int start, int end) =>
                    start < line.Length ? line[start..Math.Min(end < 0 ? line.Length : end, line.Length)].Trim() : "";

                var name = Slice(cName, cId);
                var id   = Slice(cId, cVer);
                var cur  = Slice(cVer, cAvail);
                var avail= Slice(cAvail, cSrc);
                // A real winget row has a non-empty package Id with no spaces (Google.Chrome,
                // Microsoft.VisualStudio…) and a known current version — anything else is a stray
                // wrapped/footer line, so skip it.
                if (name.Length == 0 || id.Length == 0 || id.Contains(' ')) continue;
                if (cur.Length == 0 || cur.Equals("Unknown", StringComparison.OrdinalIgnoreCase)) continue;
                list.Add(new OutdatedApp(name, id, cur, avail));
            }
        }
        catch { }
        return list;
    }
}
```
