---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\StatusColors.cs
---

# PartnerTool\StatusColors.cs

```csharp
using System.Windows.Media;

namespace PartnerTool;

public static class StatusColors
{
    // Frozen so WPF can share them freely (and cheaply) across the UI and any thread.
    public static readonly SolidColorBrush Green  = Frozen(0xA6, 0xE3, 0xA1);
    public static readonly SolidColorBrush Red    = Frozen(0xF3, 0x8B, 0xA8);
    public static readonly SolidColorBrush Yellow = Frozen(0xF9, 0xE2, 0xAF);
    public static readonly SolidColorBrush Blue   = Frozen(0x89, 0xB4, 0xFA);
    public static readonly SolidColorBrush Muted  = Frozen(0x6C, 0x70, 0x86);
    public static readonly SolidColorBrush Text   = Frozen(0xCD, 0xD6, 0xF4);   // standard foreground

    private static SolidColorBrush Frozen(byte r, byte g, byte b)
    {
        var brush = new SolidColorBrush(Color.FromRgb(r, g, b));
        brush.Freeze();
        return brush;
    }
}
```
