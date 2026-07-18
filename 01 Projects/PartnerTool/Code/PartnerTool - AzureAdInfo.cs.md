---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\AzureAdInfo.cs
---

# PartnerTool\AzureAdInfo.cs

```csharp
using System.Text.RegularExpressions;

namespace PartnerTool;

/// <summary>
/// Device join state from <c>dsregcmd /status</c> — Azure AD joined / hybrid / on-prem
/// domain joined, tenant and device id. Increasingly the first thing to check on modern
/// client machines (sign-in / Intune / conditional-access issues).
/// </summary>
public class AzureAdInfo
{
    public bool   AzureAdJoined  { get; set; }
    public bool   DomainJoined   { get; set; }
    public bool   EnterpriseJoined { get; set; }
    public string TenantName     { get; set; } = "—";
    public string DeviceId       { get; set; } = "—";

    public static async Task<AzureAdInfo> CollectAsync()
    {
        var info = new AzureAdInfo();
        try
        {
            var text = await ProcessRunner.RunCaptureAsync("dsregcmd.exe", "/status", 15000);
            bool Yes(string key)
            {
                var m = Regex.Match(text, $@"{Regex.Escape(key)}\s*:\s*(YES|NO)", RegexOptions.IgnoreCase);
                return m.Success && m.Groups[1].Value.Equals("YES", StringComparison.OrdinalIgnoreCase);
            }
            string Val(string key)
            {
                var m = Regex.Match(text, $@"{Regex.Escape(key)}\s*:\s*(.+)", RegexOptions.IgnoreCase);
                return m.Success ? m.Groups[1].Value.Trim() : "—";
            }
            info.AzureAdJoined    = Yes("AzureAdJoined");
            info.DomainJoined     = Yes("DomainJoined");
            info.EnterpriseJoined = Yes("EnterpriseJoined");
            info.TenantName       = Val("TenantName");
            info.DeviceId         = Val("DeviceId");
        }
        catch { }
        return info;
    }
}
```
