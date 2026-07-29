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
    public WuPolicyInfo?                WuPolicy   { get; init; }
    public List<ShareInfo>              Shares     { get; init; } = new();
    public CrashInfo?                   Crashes    { get; init; }
    public BootPerfInfo?                BootPerf   { get; init; }
    public List<PendingUpdate>          Pending    { get; init; } = new();
    public List<OutdatedApp>            AppUpdates { get; init; } = new();
    public List<RestorePoint>           Restore    { get; init; } = new();
    public List<UserProfileItem>        Profiles   { get; init; } = new();
    public List<OptionalFeature>        Features   { get; init; } = new();

    // Deliberately NOT here: BitLocker recovery keys. The bundle gets attached to tickets, so the
    // keys stay on the Security page where a tech reads them off screen.
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
        var wuPolicy  = Task.Run(WuPolicyInfo.Collect);
        var shares    = Task.Run(SharesInfo.Collect);
        var crashes   = Task.Run(CrashInfo.Collect);
        var bootPerf  = Task.Run(BootPerfInfo.Collect);
        var pending   = Task.Run(PendingUpdatesInfo.Collect);
        var appUpd    = OutdatedAppsInfo.CollectAsync();
        var restore   = Task.Run(RestorePointsInfo.Collect);
        var profiles  = Task.Run(SystemManagement.UserProfiles);
        var features  = Task.Run(SystemManagement.OptionalFeatures);

        await Task.WhenAll(software, startup, adapters, harden, wifi,
                           prosentry, defender, services, drivers, tasks, envVars, hosts, wuPolicy,
                           shares, crashes, bootPerf, pending, appUpd, restore, profiles, features);

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
            WuPolicy  = await wuPolicy,
            Shares    = await shares,
            Crashes   = await crashes,
            BootPerf  = await bootPerf,
            Pending   = await pending,
            AppUpdates= await appUpd,
            Restore   = await restore,
            Profiles  = await profiles,
            Features  = await features,
        };
    }
}
```
