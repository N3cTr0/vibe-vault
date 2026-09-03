---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\RoundsConfig.cs
---

# src\Octavia.Core\Core\RoundsConfig.cs

```csharp
namespace Octavia.Core;

/// What she checks on her own, and when she is allowed to say so.
///
/// A whole object rather than four loose fields, because these only mean anything together:
/// an interval with no quiet hours is a doorbell at four in the morning, and quiet hours with
/// nothing switched on are decoration. See `Watchman`.
internal sealed class RoundsConfig
{
    /// Whether she walks her rounds at all. On by default, which is safe while no round
    /// exists that can speak — and the moment one does, this is the switch that stops it
    /// without editing code.
    public bool Enabled { get; set; } = true;

    /// Minutes between walks. Clamped to a day at the top and a minute at the bottom, so a
    /// mistyped zero is a busy loop that never happens.
    public int EveryMinutes { get; set; } = 60;

    /// When she stays silent. Findings inside this window are **held, not dropped** — the
    /// point of checking at four in the morning is that somebody hears about it at eight.
    ///
    /// `From` equal to `To` means no quiet hours at all, which is the honest way to switch
    /// them off: a window of zero length rather than a magic empty string.
    public string QuietFrom { get; set; } = "22:30";
    public string QuietTo { get; set; } = "07:30";
}
```
