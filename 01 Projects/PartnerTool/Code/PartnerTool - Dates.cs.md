---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Dates.cs
---

# PartnerTool\Dates.cs

```csharp
using System.Globalization;

namespace PartnerTool;

/// <summary>
/// Canonical date/time formatting for everything the tool shows a tech.
///
/// **Progressive Computing standard: displayed dates are always MM/DD/YYYY.** Use these format
/// strings (or the helpers) for every user-facing date so the whole app reads consistently — never
/// hand-roll a `d MMM yyyy` / `dd/MM/yy` / locale-default format. The app also pins the process
/// culture to en-US with a MM/dd/yyyy short-date pattern (see <c>App.ForceEnUsCulture</c>), which
/// covers any default <c>DateTime.ToString()</c> and XAML <c>{0:d}</c> binding as a safety net.
///
/// XAML bindings can't reference these consts, so they use the same literal patterns inline, e.g.
/// <c>StringFormat={}{0:MM/dd/yyyy}</c> / <c>{0:MM/dd/yyyy HH:mm}</c> — keep them in sync with here.
/// </summary>
public static class Dates
{
    /// <summary>MM/DD/YYYY — the house date format.</summary>
    public const string Date = "MM/dd/yyyy";
    /// <summary>MM/DD/YYYY HH:mm.</summary>
    public const string DateTime = "MM/dd/yyyy HH:mm";
    /// <summary>MM/DD/YYYY HH:mm:ss.</summary>
    public const string DateTimeSec = "MM/dd/yyyy HH:mm:ss";

    public static string Format(DateTime dt) => dt.ToString(Date, CultureInfo.InvariantCulture);
    public static string FormatWithTime(DateTime dt) => dt.ToString(DateTime, CultureInfo.InvariantCulture);

    /// <summary>
    /// Reformat a date string that came from an external tool (e.g. <c>schtasks</c>, whose output
    /// follows the machine's locale) into MM/DD/YYYY. Non-dates ("N/A", "Disabled", blank) pass
    /// through untouched, as does anything that won't parse.
    /// </summary>
    public static string ReformatOrKeep(string? raw)
    {
        if (string.IsNullOrWhiteSpace(raw)) return raw ?? "";
        if (!System.DateTime.TryParse(raw, CultureInfo.CurrentCulture, DateTimeStyles.None, out var dt) &&
            !System.DateTime.TryParse(raw, CultureInfo.InvariantCulture, DateTimeStyles.None, out dt))
            return raw;
        return dt.TimeOfDay == TimeSpan.Zero ? Format(dt) : FormatWithTime(dt);
    }
}
```
