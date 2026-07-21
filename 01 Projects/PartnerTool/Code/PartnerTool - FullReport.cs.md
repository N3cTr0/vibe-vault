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
    public required SystemSnapshot      Snap       { get; init; }
    public List<InstalledApp>           Software   { get; init; } = new();
    public List<StartupEntry>           Startup    { get; init; } = new();
    public List<AdapterSummary>         Adapters   { get; init; } = new();
    public List<string>                 Wifi       { get; init; } = new();
    public List<AuditItem>              Hardening  { get; init; } = new();
    public ProsentryReport?             Prosentry  { get; init; }
    public DefenderInfo?                Defender   { get; init; }
    public List<ServiceItem>            Services   { get; init; } = new();
    public List<DriverItem>             Drivers    { get; init; } = new();
    public List<ScheduledTaskItem>      Tasks      { get; init; } = new();
    public List<EnvVar>                 EnvVars    { get; init; } = new();
    public string                       HostsFile  { get; init; } = "";
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
        // Everything the Manage/Security tabs surface, so the bundle is a full as-found record.
        var prosentry = Task.Run(ProsentryInfo.Collect);
        var defender  = Task.Run(DefenderInfo.Collect);
        var services  = Task.Run(ServicesInfo.Collect);
        var drivers   = Task.Run(DriversInfo.Collect);
        var tasks     = ScheduledTasksInfo.CollectAsync();
        var envVars   = Task.Run(SystemManagement.EnvVars);
        var hosts     = Task.Run(SystemManagement.ReadHosts);

        await Task.WhenAll(software, startup, adapters, harden, wifi,
                           prosentry, defender, services, drivers, tasks, envVars, hosts);

        return new FullReportData
        {
            Snap      = snap,
            Software  = await software,
            Startup   = await startup,
            Adapters  = await adapters,
            Hardening = await harden,
            Wifi      = await wifi,
            Prosentry = await prosentry,
            Defender  = await defender,
            Services  = await services,
            Drivers   = await drivers,
            Tasks     = await tasks,
            EnvVars   = await envVars,
            HostsFile = await hosts,
        };
    }
}
```
