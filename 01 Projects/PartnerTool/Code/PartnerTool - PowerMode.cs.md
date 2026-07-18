---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PowerMode.cs
---

# PartnerTool\PowerMode.cs

```csharp
using System.Runtime.InteropServices;

namespace PartnerTool;

public record PowerModeOption(string Name, Guid Guid, bool Active);

/// <summary>
/// Windows 11 "Power Mode" (the Settings slider: Best power efficiency / Balanced / Best
/// performance). On modern machines there's only one legacy power scheme (Balanced) and this
/// overlay is the real control — set via the powrprof.dll overlay APIs the Settings UI uses.
/// </summary>
public static class PowerMode
{
    public static readonly Guid Efficiency  = new("961cc777-2547-4f9d-8174-7d86181b8a7a");
    public static readonly Guid Balanced    = new("00000000-0000-0000-0000-000000000000");
    public static readonly Guid Performance = new("ded574b5-45a0-4f42-8737-46345c09c238");

    // PowerGetActiveOverlayScheme isn't exported on all builds; PowerGetEffectiveOverlayScheme
    // is and returns the mode actually in effect, which is what we want to display.
    [DllImport("powrprof.dll")] private static extern uint PowerGetEffectiveOverlayScheme(out Guid overlayGuid);
    [DllImport("powrprof.dll")] private static extern uint PowerSetActiveOverlayScheme(Guid overlayGuid);

    public static Guid? Active()
    {
        try { return PowerGetEffectiveOverlayScheme(out var g) == 0 ? g : null; }
        catch { return null; }
    }

    public static bool Supported() => Active() is not null;

    public static string ActiveName() => Name(Active());

    private static string Name(Guid? g) => g switch
    {
        { } x when x == Efficiency  => "Best power efficiency",
        { } x when x == Performance => "Best performance",
        { } x when x == Balanced    => "Balanced",
        _ => "Unknown",
    };

    public static List<PowerModeOption> List()
    {
        var active = Active();
        return new List<PowerModeOption>
        {
            new("Best power efficiency", Efficiency,  active == Efficiency),
            new("Balanced",              Balanced,    active == Balanced),
            new("Best performance",      Performance, active == Performance),
        };
    }

    public static void Set(Guid g)
    {
        try { PowerSetActiveOverlayScheme(g); } catch { }
    }
}
```
