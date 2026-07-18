---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TechGate.cs
---

# PartnerTool\TechGate.cs

```csharp
using System.Windows;

namespace PartnerTool;

/// <summary>
/// Soft gate in front of destructive actions: the first time a tech triggers one in a session they
/// must enter today's date code — day-of-month then 4-digit year (e.g. 6/26/2026 → "262026"). Once
/// entered correctly the session is unlocked and later destructive actions go straight to their own
/// confirm. It's a "are you really a tech" speed-bump to stop an end-user clicking something
/// destructive, not real security.
///
/// The unlock expires after <see cref="Timeout"/> of no gated actions. Techs routinely leave the app
/// (e.g. the Update All window) open and unattended on a client's desk, so an unlock that lasted the
/// whole process lifetime would hand the machine to whoever sits down next. Every successful check
/// slides the window forward, so an actively-working tech is never re-prompted mid-job.
///
/// The lock indicator in the window header (<c>MainWindow.BtnGate</c>) shows the state and the
/// remaining time, and lets a tech lock out by hand before walking away.
/// </summary>
public static class TechGate
{
    /// <summary>How long an unlock survives with no gated action before the code is required again.</summary>
    public static readonly TimeSpan Timeout = TimeSpan.FromMinutes(15);

    private static DateTime _verifiedUntil = DateTime.MinValue;

    /// <summary>Raised when the gate locks or unlocks, so the header indicator can repaint at once.</summary>
    public static event Action? Changed;

    /// <summary>Today's code: day (no leading zero) followed by the 4-digit year.</summary>
    public static string TodayCode => $"{DateTime.Now.Day}{DateTime.Now.Year}";

    /// <summary>True while the session is unlocked and the idle timeout hasn't lapsed.</summary>
    public static bool IsUnlocked => DateTime.UtcNow < _verifiedUntil;

    /// <summary>Time left on the current unlock; <see cref="TimeSpan.Zero"/> when locked.</summary>
    public static TimeSpan Remaining
    {
        get
        {
            var left = _verifiedUntil - DateTime.UtcNow;
            return left > TimeSpan.Zero ? left : TimeSpan.Zero;
        }
    }

    /// <summary>Drops the unlock immediately (the header lock button, or a tech stepping away).</summary>
    public static void Lock()
    {
        if (!IsUnlocked) return;
        _verifiedUntil = DateTime.MinValue;
        ActivityLog.Action("TechGate", "Locked by tech");
        Changed?.Invoke();
    }

    /// <summary>True if the tech is already verified or enters the code now; false if they cancel/fail.</summary>
    public static bool Verify(Window? owner)
    {
        if (IsUnlocked) { Slide(); return true; }

        var expired = _verifiedUntil != DateTime.MinValue;   // unlocked earlier, then timed out
        var w = new TechGateWindow(expired) { Owner = owner };
        if (w.ShowDialog() != true) return false;

        Slide();
        ActivityLog.Action("TechGate", $"Unlocked (expires after {Timeout.TotalMinutes:0} min idle)");
        return true;
    }

    private static void Slide()
    {
        _verifiedUntil = DateTime.UtcNow + Timeout;
        Changed?.Invoke();
    }
}
```
