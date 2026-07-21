---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SecuritySoftware.cs
---

# PartnerTool\SecuritySoftware.cs

```csharp
namespace PartnerTool;

/// <summary>
/// Single source of truth for "is this AV / EDR / security software?" — used to block the tool from
/// weakening a machine's protection: stopping its services, killing its processes, disabling its
/// startup entries, or uninstalling it. Match on any name/command/display-name string.
/// </summary>
public static class SecuritySoftware
{
    // Keep lowercase. Substrings are matched against a lowercased haystack, so keep them broad
    // enough to catch product/service/exe/display-name variants but specific enough not to snag
    // unrelated software.
    private static readonly string[] Keywords =
    {
        "defender", "securityhealth", "windows security",
        "sentinelone", "sentinel one", "sentinelagent",
        "huntress", "crowdstrike", "falcon", "cylance", "carbon black", "carbonblack",
        "sophos", "bitdefender", "webroot", "malwarebytes", "kaspersky",
        // ESET — bare "eset" would substring-match "Reset", so use its product/exe/service names.
        "eset nod32", "eset endpoint", "eset internet", "eset smart", "eset security",
        "eset file", "eset server", "eset mail", "ekrn", "egui",
        "mcafee", "trellix", "cortex xdr", "symantec", "norton", "avast", "avg antivirus",
        "trend micro", "vipre", "threatlocker", "blackpoint", "todyl",
    };

    /// <summary>True if the text names an AV/EDR/security product (case-insensitive substring match).</summary>
    public static bool Matches(string? text)
    {
        if (string.IsNullOrWhiteSpace(text)) return false;
        var hay = text.ToLowerInvariant();
        return Keywords.Any(hay.Contains);
    }
}
```
