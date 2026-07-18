---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\DefenderInfo.cs
---

# PartnerTool\DefenderInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

/// <summary>
/// Microsoft Defender status from the root\Microsoft\Windows\Defender WMI namespace —
/// real-time protection, signature age, last scans and threat count. Returns
/// <see cref="Available"/> = false when Defender isn't the active AV (e.g. a third-party
/// product is installed), in which case the page just says so.
/// </summary>
public class DefenderInfo
{
    public bool      Available                 { get; set; }
    public bool      RealTimeProtection        { get; set; }
    public bool      AntivirusEnabled          { get; set; }
    public bool      TamperProtection          { get; set; }
    public string    SignatureVersion          { get; set; } = "—";
    public DateTime? SignatureUpdated          { get; set; }
    public int?      QuickScanAgeDays          { get; set; }
    public int?      FullScanAgeDays           { get; set; }
    public int       ThreatCount               { get; set; }

    public static DefenderInfo Collect()
    {
        var d = new DefenderInfo();
        try
        {
            using var q = new ManagementObjectSearcher(@"root\Microsoft\Windows\Defender",
                "SELECT * FROM MSFT_MpComputerStatus");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                d.Available          = true;
                d.RealTimeProtection = o["RealTimeProtectionEnabled"] is bool rtp && rtp;
                d.AntivirusEnabled   = o["AntivirusEnabled"] is bool av && av;
                d.TamperProtection   = o["IsTamperProtected"] is bool tp && tp;
                d.SignatureVersion   = o["AntivirusSignatureVersion"]?.ToString() ?? "—";
                try { if (o["AntivirusSignatureLastUpdated"] != null)
                        d.SignatureUpdated = ManagementDateTimeConverter.ToDateTime(o["AntivirusSignatureLastUpdated"].ToString()); }
                catch { }
                if (o["QuickScanAge"] is { } qa && long.TryParse(qa.ToString(), out var qd) && qd < 100000) d.QuickScanAgeDays = (int)qd;
                if (o["FullScanAge"]  is { } fa && long.TryParse(fa.ToString(), out var fd) && fd < 100000) d.FullScanAgeDays  = (int)fd;
                break;
            }
        }
        catch { }

        if (d.Available)
        {
            try
            {
                using var q = new ManagementObjectSearcher(@"root\Microsoft\Windows\Defender",
                    "SELECT ThreatID FROM MSFT_MpThreat");
                d.ThreatCount = q.Get().Count;
            }
            catch { }
        }
        return d;
    }
}
```
