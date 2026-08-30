---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Core\Hotkey.cs
---

# src\Octavia.App\Core\Hotkey.cs

```csharp
using System.Globalization;

namespace Octavia.Core;

internal readonly record struct Hotkey(uint Modifiers, uint VirtualKey, string Display)
{
    private const uint ModAlt = 0x0001, ModControl = 0x0002, ModShift = 0x0004, ModWin = 0x0008;
    private const uint ModNoRepeat = 0x4000;

    public static bool TryParse(string? text, out Hotkey hotkey)
    {
        hotkey = default;
        if (string.IsNullOrWhiteSpace(text)) return false;

        var parts = text.Split('+', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
        if (parts.Length == 0) return false;

        uint modifiers = ModNoRepeat;
        uint? key = null;

        foreach (var part in parts)
        {
            switch (part.ToUpperInvariant())
            {
                case "CTRL" or "CONTROL": modifiers |= ModControl; break;
                case "ALT": modifiers |= ModAlt; break;
                case "SHIFT": modifiers |= ModShift; break;
                case "WIN": modifiers |= ModWin; break;
                default:
                    key = VirtualKeyFor(part);
                    if (key is null) return false;
                    break;
            }
        }

        if (key is null || (modifiers & (ModControl | ModAlt | ModShift | ModWin)) == 0) return false;

        hotkey = new Hotkey(modifiers, key.Value, Normalise(parts));
        return true;
    }

    private static uint? VirtualKeyFor(string token)
    {
        var upper = token.ToUpperInvariant();

        if (upper.Length == 1 && (char.IsAsciiLetterUpper(upper[0]) || char.IsAsciiDigit(upper[0])))
            return upper[0];

        if (upper.StartsWith('F') && upper.Length is 2 or 3 &&
            int.TryParse(upper[1..], NumberStyles.None, CultureInfo.InvariantCulture, out var n) &&
            n is >= 1 and <= 24)
            return (uint)(0x70 + n - 1);

        return upper switch
        {
            "SPACE" => 0x20,
            "TAB" => 0x09,
            "ENTER" or "RETURN" => 0x0D,
            "ESC" or "ESCAPE" => 0x1B,
            "INSERT" => 0x2D,
            "HOME" => 0x24,
            "END" => 0x23,
            "PAGEUP" => 0x21,
            "PAGEDOWN" => 0x22,
            "`" or "BACKTICK" or "GRAVE" => 0xC0,
            _ => null
        };
    }

    private static string Normalise(IEnumerable<string> parts) =>
        string.Join('+', parts.Select(p => p.Length == 1
            ? p.ToUpperInvariant()
            : string.Concat(char.ToUpperInvariant(p[0]), p[1..].ToLowerInvariant())));
}
```
