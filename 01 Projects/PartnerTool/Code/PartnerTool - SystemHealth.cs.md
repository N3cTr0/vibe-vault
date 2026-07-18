---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SystemHealth.cs
---

# PartnerTool\SystemHealth.cs

```csharp
using Microsoft.Win32;

namespace PartnerTool;

public static class SystemHealth
{
    /// <summary>
    /// True when Windows genuinely needs a restart to finish servicing — Component-Based
    /// Servicing flagged a reboot, or a Windows Update installed something that needs one.
    ///
    /// Deliberately does NOT check PendingFileRenameOperations. That value is written by
    /// all sorts of routine software (OneDrive/antivirus/installer cleanups scheduling a
    /// file swap on next boot) and is almost always present on a healthy machine, so it
    /// produces constant false "reboot pending" positives. Don't re-add it.
    /// </summary>
    public static bool IsRebootPending() => PendingReasons().Count > 0;

    /// <summary>
    /// Why a restart is pending, if any — so the UI can show the cause rather than a bare
    /// "reboot pending". Empty list means nothing is pending. Same two sources as before
    /// (CBS + Windows Update); still deliberately ignores PendingFileRenameOperations.
    /// </summary>
    public static List<string> PendingReasons()
    {
        var reasons = new List<string>();
        try
        {
            using (var k = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"))
                if (k != null) reasons.Add("Windows servicing (Component-Based Servicing)");

            using (var k = Registry.LocalMachine.OpenSubKey(
                @"SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired"))
                if (k != null) reasons.Add("Windows Update");
        }
        catch { }
        return reasons;
    }
}
```
