---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ServicingLock.cs
---

# PartnerTool\ServicingLock.cs

```csharp
namespace PartnerTool;

/// <summary>
/// App-wide gate for heavy Windows-servicing operations (DISM / SFC / CHKDSK / Windows Update
/// work). The Repair page and the Updates page each have their own busy state, so nothing used to
/// stop a tech running Full Repair and Update All at the same time — both drive the same CBS /
/// TrustedInstaller stack and the loser fails (observed in the field: DISM RestoreHealth died with
/// 0x800F0915 while Update All's PSWindowsUpdate install was running). Acquire this before any
/// servicing-heavy operation; the second caller gets a friendly "wait for X to finish".
/// </summary>
public static class ServicingLock
{
    private static readonly object Gate = new();

    /// <summary>Human-readable name of the operation currently holding the lock, or null.</summary>
    public static string? CurrentOperation { get; private set; }

    public static bool TryAcquire(string operation)
    {
        lock (Gate)
        {
            if (CurrentOperation != null) return false;
            CurrentOperation = operation;
            return true;
        }
    }

    public static void Release()
    {
        lock (Gate) CurrentOperation = null;
    }
}
```
