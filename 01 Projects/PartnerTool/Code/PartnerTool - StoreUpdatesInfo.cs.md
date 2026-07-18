---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\StoreUpdatesInfo.cs
---

# PartnerTool\StoreUpdatesInfo.cs

```csharp
using Windows.ApplicationModel.Store.Preview.InstallControl;

namespace PartnerTool;

/// <summary>
/// Lists pending Microsoft Store (UWP/MSIX) app updates without installing them. Uses the same
/// AppInstallManager the Update All flow uses, but passes AppUpdateOptions with
/// AutomaticallyDownloadAndInstallUpdateIfFound = false so this stays a read-only scan — it must
/// not kick off downloads (or cancel updates Windows already started) just from opening the tab.
///
/// A fresh SearchForAllUpdatesAsync does NOT return updates the Store has already queued itself
/// (e.g. blocked with 0x80073D02 "app was in use", or sitting at ReadyToDownload) — those live in
/// the AppInstallItems queue instead, so the scan reads both and merges.
/// </summary>
public static class StoreUpdatesInfo
{
    /// <returns>Pending update lines ("name — state"), or <c>null</c> when the Store can't be
    /// queried at all (no Store on LTSC/Server, service disabled, …) — so the UI can say
    /// "unavailable" instead of the misleading "no pending updates".</returns>
    public static async Task<List<string>?> ScanAsync()
    {
        try
        {
            var mgr = new AppInstallManager();
            var found = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);

            // Already-queued items first (a live install state beats "update available").
            try
            {
                foreach (var item in mgr.AppInstallItemsWithGroupSupport)
                {
                    if (item.InstallType != AppInstallType.Update) continue;
                    var state = DescribePending(item);
                    if (state != null) found[Name(item.PackageFamilyName)] = state;
                }
            }
            catch { /* queue unreadable — fall back to search only */ }

            var options = new AppUpdateOptions { AutomaticallyDownloadAndInstallUpdateIfFound = false };
            var updates = await mgr.SearchForAllUpdatesAsync(string.Empty, string.Empty, options);
            foreach (var u in updates)
                found.TryAdd(Name(u.PackageFamilyName), "update available");

            return found.OrderBy(kv => kv.Key, StringComparer.OrdinalIgnoreCase)
                        .Select(kv => $"{kv.Key} — {kv.Value}")
                        .ToList();
        }
        catch { return null; }   // Store unavailable / not signed in
    }

    private static string Name(string pfn) =>
        string.IsNullOrEmpty(pfn) ? "(unknown)" : pfn.Split('_')[0];

    /// <returns>A short state description, or <c>null</c> when the item is no longer pending.</returns>
    private static string? DescribePending(AppInstallItem item)
    {
        AppInstallStatus status;
        try { status = item.GetCurrentStatus(); }
        catch { return "queued"; }

        const int PackagesInUse = unchecked((int)0x80073D02);   // ERROR_PACKAGES_IN_USE
        return status.InstallState switch
        {
            AppInstallState.Completed or AppInstallState.Canceled => null,
            AppInstallState.Error when status.ErrorCode?.HResult == PackagesInUse
                => "waiting — app was in use, retries when it closes",
            AppInstallState.Error
                => $"failed (0x{status.ErrorCode?.HResult:X8}) — Update All retries it",
            AppInstallState.Downloading => $"downloading {status.PercentComplete}%",
            AppInstallState.Installing  => "installing",
            _ => "queued",
        };
    }
}
```
