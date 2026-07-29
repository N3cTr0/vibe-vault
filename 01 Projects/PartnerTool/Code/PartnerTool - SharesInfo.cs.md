---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SharesInfo.cs
---

# PartnerTool\SharesInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record ShareInfo(string Name, string Path, string Description, string Kind, bool Admin)
{
    public string Detail => string.Join("   ·   ",
        new[] { Kind, Path, Description }.Where(s => !string.IsNullOrWhiteSpace(s)));
}

/// <summary>
/// Shared folders on this PC via Win32_Share, including the hidden administrative shares
/// (C$, ADMIN$, IPC$) - the high bit of Type marks those.
/// </summary>
public static class SharesInfo
{
    private const uint AdminFlag = 0x80000000;

    public static List<ShareInfo> Collect()
    {
        var list = new List<ShareInfo>();
        try
        {
            using var q = new ManagementObjectSearcher("SELECT Name, Path, Description, Type FROM Win32_Share");
            foreach (ManagementObject o in q.Get())
                using (o)
                {
                    uint type  = Convert.ToUInt32(o["Type"] ?? 0u);
                    bool admin = (type & AdminFlag) != 0;
                    string kind = (type & ~AdminFlag) switch
                    {
                        0 => "Folder",
                        1 => "Printer",
                        2 => "Device",
                        3 => "IPC",
                        _ => "Other",
                    };
                    list.Add(new ShareInfo(
                        o["Name"]?.ToString() ?? "",
                        o["Path"]?.ToString() ?? "",
                        o["Description"]?.ToString() ?? "",
                        admin ? $"{kind} (admin)" : kind,
                        admin));
                }
        }
        catch { }
        // Real shares first, then the hidden admin ones.
        return list.OrderBy(s => s.Admin)
                   .ThenBy(s => s.Name, StringComparer.OrdinalIgnoreCase)
                   .ToList();
    }
}
```
