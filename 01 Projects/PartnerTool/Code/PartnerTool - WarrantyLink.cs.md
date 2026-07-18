---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\WarrantyLink.cs
---

# PartnerTool\WarrantyLink.cs

```csharp
namespace PartnerTool;

/// <summary>
/// Builds a manufacturer warranty-lookup URL from the machine's serial number. Dell and
/// Lenovo accept the serial directly in the URL (true deep link); for everyone else we
/// open the vendor's warranty page and rely on the serial being copied to the clipboard.
/// </summary>
public static class WarrantyLink
{
    /// <summary>Best warranty-lookup URL for this manufacturer + serial.</summary>
    public static string For(string manufacturer, string serial)
    {
        var m = (manufacturer ?? "").ToLowerInvariant();
        var s = Uri.EscapeDataString((serial ?? "").Trim());

        if (m.Contains("dell"))
            return $"https://www.dell.com/support/home/en-us/product-support/servicetag/{s}/overview";
        if (m.Contains("lenovo"))
            return $"https://pcsupport.lenovo.com/us/en/warrantylookup?serialNumber={s}";
        if (m.Contains("hewlett") || m.Contains("hp"))
            return "https://support.hp.com/us-en/check-warranty";
        if (m.Contains("microsoft"))
            return "https://account.microsoft.com/devices";
        if (m.Contains("asus"))
            return "https://www.asus.com/support/warranty-status-inquiry";
        if (m.Contains("acer"))
            return "https://www.acer.com/us-en/support/warranty-find";

        // Unknown OEM — a serial-scoped web search is the most useful fallback.
        return $"https://www.bing.com/search?q={Uri.EscapeDataString($"{manufacturer} warranty check {serial}")}";
    }

    /// <summary>True when the URL already contains the serial (no manual paste needed).</summary>
    public static bool IsPrefilled(string manufacturer)
    {
        var m = (manufacturer ?? "").ToLowerInvariant();
        return m.Contains("dell") || m.Contains("lenovo");
    }
}
```
