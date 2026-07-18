---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\FullReport.cs
---

# PartnerTool\FullReport.cs

```csharp
namespace PartnerTool;

/// <summary>
/// Everything the tool can collect, gathered for the diagnostics report: the startup
/// <see cref="SystemSnapshot"/> plus the on-demand inventories that aren't in it (installed
/// software, startup programs, every network adapter, saved Wi-Fi networks, the hardening
/// scorecard).
///
/// GOING FORWARD: when a new collector is added to the tool, surface it here (a field below +
/// a line in <see cref="FullReport.GatherAsync"/>) and add a section to <see cref="ReportBuilder"/>
/// so it shows up in the Collect Diagnostics report.
/// </summary>
public class FullReportData
{
    public required SystemSnapshot    Snap      { get; init; }
    public List<InstalledApp>         Software  { get; init; } = new();
    public List<StartupEntry>         Startup   { get; init; } = new();
    public List<AdapterSummary>       Adapters  { get; init; } = new();
    public List<string>              Wifi       { get; init; } = new();
    public List<AuditItem>            Hardening { get; init; } = new();
}

public static class FullReport
{
    public static async Task<FullReportData> GatherAsync(SystemSnapshot snap)
    {
        var software = Task.Run(SoftwareInventory.Collect);
        var startup  = Task.Run(StartupInfo.Collect);
        var adapters = Task.Run(NetworkInfo.AllAdapters);
        var harden   = Task.Run(SecurityAudit.Collect);
        var wifi     = WifiInfo.GetProfilesAsync();

        await Task.WhenAll(software, startup, adapters, harden, wifi);

        return new FullReportData
        {
            Snap      = snap,
            Software  = await software,
            Startup   = await startup,
            Adapters  = await adapters,
            Hardening = await harden,
            Wifi      = await wifi,
        };
    }
}
```
