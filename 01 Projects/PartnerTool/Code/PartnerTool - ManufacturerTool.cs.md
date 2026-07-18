---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ManufacturerTool.cs
---

# PartnerTool\ManufacturerTool.cs

```csharp
using System.IO;

namespace PartnerTool;

public record ManufacturerTool(
    string   DisplayName,
    string   Description,
    string[] ExePaths,
    string?  StoreUri = null
)
{
    public string? ResolveExe() => ExePaths.FirstOrDefault(File.Exists);
}

public static class ManufacturerTools
{
    private static readonly Dictionary<string, ManufacturerTool> Lookup =
        new(StringComparer.OrdinalIgnoreCase)
    {
        ["dell"] = new(
            "Dell Command Update",
            "Update Dell drivers, firmware, and BIOS.",
            [
                @"C:\Program Files\Dell\CommandUpdate\DCUx86.exe",
                @"C:\Program Files (x86)\Dell\CommandUpdate\DCUx86.exe",
                @"C:\Program Files\Dell\CommandUpdate\dcu-cli.exe",
            ]
        ),
        ["lenovo"] = new(
            "Lenovo Vantage",
            "Update Lenovo drivers, firmware, and system software.",
            [
                @"C:\Program Files (x86)\Lenovo\System Update\tvsu.exe",
                @"C:\Program Files\Lenovo\Vantage\LenovoVantage.exe",
            ],
            StoreUri: "ms-windows-store://pdp/?productid=9WZDNCRFJ4MV"
        ),
        ["hp"] = new(
            "HP Support Assistant",
            "Update HP drivers, firmware, and software.",
            [
                @"C:\Program Files (x86)\HP\HP Support Framework\HPSF.exe",
                @"C:\Program Files\HP\HP Support Framework\HPSF.exe",
                @"C:\Program Files (x86)\HP\HP Support Assistant\HPSAApp.exe",
            ]
        ),
        ["hewlett"] = new(   // catches "Hewlett-Packard"
            "HP Support Assistant",
            "Update HP drivers, firmware, and software.",
            [
                @"C:\Program Files (x86)\HP\HP Support Framework\HPSF.exe",
                @"C:\Program Files\HP\HP Support Framework\HPSF.exe",
            ]
        ),
        ["asus"] = new(
            "MyASUS",
            "Update ASUS drivers and system software.",
            [
                @"C:\Program Files\ASUS\MyASUS\lib\MyASUS.exe",
            ],
            StoreUri: "ms-windows-store://pdp/?productid=9N7R5S6B0ZHH"
        ),
        ["acer"] = new(
            "Acer Care Center",
            "Update Acer drivers and software.",
            [
                @"C:\ProgramData\Acer\Care Center\carecenter.exe",
                @"C:\Program Files\Acer Care Center\carecenter.exe",
            ]
        ),
        ["samsung"] = new(
            "Samsung Update",
            "Update Samsung drivers and software.",
            [
                @"C:\Program Files (x86)\Samsung\Samsung Update\SamsungUpdate.exe",
            ]
        ),
        ["microsoft"] = new(   // Surface devices
            "Surface App",
            "Update Surface firmware and hardware components.",
            [],
            StoreUri: "ms-windows-store://pdp/?productid=9PBZBG6NW0KD"
        ),
        ["toshiba"] = new(
            "Toshiba Service Station",
            "Update Toshiba drivers and software.",
            [
                @"C:\Program Files (x86)\TOSHIBA\Service Station\ToshibaServiceStation.exe",
            ]
        ),
    };

    // Model/manufacturer substrings that mean "this is a virtual machine, not real hardware".
    private static readonly string[] VmSignatures =
    {
        "virtual machine", "hyper-v", "cloud pc", "vmware", "virtualbox", "innotek",
        "kvm", "qemu", "xen", "bochs", "parallels", "google compute engine", "amazon ec2",
    };

    /// <summary>True for VMs — Windows 365 Cloud PC, Azure/Hyper-V, VMware, VirtualBox, etc.
    /// They have no OEM firmware/driver tool, so we must not offer one.</summary>
    public static bool IsVirtualMachine(string manufacturer, string model)
    {
        var hay = ((manufacturer ?? "") + " " + (model ?? "")).ToLowerInvariant();
        return VmSignatures.Any(hay.Contains);
    }

    public static ManufacturerTool? Get(string manufacturer, string model)
    {
        manufacturer ??= ""; model ??= "";
        // A Windows 365 Cloud PC / Hyper-V VM reports Manufacturer "Microsoft Corporation" and used
        // to match the Surface tool — but a VM has no firmware to update. Skip all VMs.
        if (IsVirtualMachine(manufacturer, model)) return null;

        foreach (var (key, tool) in Lookup)
        {
            if (!manufacturer.Contains(key, StringComparison.OrdinalIgnoreCase)) continue;
            // "Microsoft" also covers non-Surface Microsoft machines — only offer the Surface tool
            // when the model actually says Surface.
            if (key == "microsoft" && !model.Contains("Surface", StringComparison.OrdinalIgnoreCase))
                return null;
            return tool;
        }
        return null;
    }
}
```
