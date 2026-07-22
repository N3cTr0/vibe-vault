---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SystemSnapshot.cs
---

# PartnerTool\SystemSnapshot.cs

```csharp
namespace PartnerTool;

/// <summary>
/// A single, point-in-time capture of everything the information tabs display.
/// Collected once at startup (behind the loading splash) so navigating between
/// tabs is instant — the pages just paint from this instead of re-querying WMI.
/// The per-page Refresh buttons still re-collect their own fresh data.
///
/// Live/heavy data that only one page needs (top processes, active connections, Wi-Fi,
/// installed-programs inventory, disk-usage scan) is deliberately NOT captured here —
/// those pages load it on demand so startup stays quick.
/// </summary>
public class SystemSnapshot
{
    public required SystemInfo        Info           { get; init; }
    public required PerfSnapshot      Perf           { get; init; }
    public required SecuritySnapshot  Security       { get; init; }
    public          AdapterSummary?   PrimaryAdapter { get; init; }
    public required HardwareInfo      Hardware       { get; init; }
    public          bool              RebootPending  { get; init; }
    public          List<string>      RebootReasons  { get; init; } = new();
    public required DiagnosticsInfo   Diagnostics    { get; init; }
    public required ReliabilityInfo   Reliability    { get; init; }
    public          List<DisplayInfo> Displays       { get; init; } = new();
    public          List<PrinterInfo> Printers       { get; init; } = new();
    public          List<LocalAccount> Accounts      { get; init; } = new();
    public required UpdateHistoryInfo Updates        { get; init; }
    public          List<PowerEvent>  PowerEvents    { get; init; } = new();
    public required SystemExtras      Extras         { get; init; }
    public required AzureAdInfo       Aad            { get; init; }
    public required PowerStatusInfo   Power          { get; init; }
    public          DateTime          CapturedAt     { get; init; }

    /// <summary>Run every collector in parallel and return the combined snapshot.</summary>
    public static async Task<SystemSnapshot> CaptureAsync()
    {
        var info     = Task.Run(SystemInfo.Collect);
        var perf     = Task.Run(PerfSnapshot.Collect);
        var sec      = Task.Run(SecuritySnapshot.Collect);
        var net      = Task.Run(NetworkInfo.GetPrimaryAdapter);
        var hw       = Task.Run(HardwareInfo.Collect);
        var reasons  = Task.Run(SystemHealth.PendingReasons);
        var diag     = Task.Run(DiagnosticsInfo.Collect);
        var rel      = Task.Run(ReliabilityInfo.CollectIndex);   // index only — records are slow, loaded on demand
        var displays = Task.Run(DisplaysInfo.Collect);
        var printers = Task.Run(PrintersInfo.Collect);
        var accounts = Task.Run(AccountsInfo.Collect);
        var updates  = Task.Run(UpdateHistoryInfo.Collect);
        var power    = Task.Run(() => PowerEventsInfo.Collect());
        var extras   = Task.Run(SystemExtras.Collect);
        var aad      = AzureAdInfo.CollectAsync();   // device join — already async (dsregcmd)
        var pwr      = PowerStatusInfo.CollectAsync();   // already async (powercfg)

        await Task.WhenAll(info, perf, sec, net, hw, reasons, diag, rel,
                           displays, printers, accounts, updates, power, extras, aad, pwr);

        var pendingReasons = await reasons;
        return new SystemSnapshot
        {
            Info           = await info,
            Perf           = await perf,
            Security       = await sec,
            PrimaryAdapter = await net,
            Hardware       = await hw,
            RebootPending  = pendingReasons.Count > 0,
            RebootReasons  = pendingReasons,
            Diagnostics    = await diag,
            Reliability    = await rel,
            Displays       = await displays,
            Printers       = await printers,
            Accounts       = await accounts,
            Updates        = await updates,
            PowerEvents    = await power,
            Extras         = await extras,
            Aad            = await aad,
            Power          = await pwr,
            CapturedAt     = DateTime.Now,
        };
    }
}
```
