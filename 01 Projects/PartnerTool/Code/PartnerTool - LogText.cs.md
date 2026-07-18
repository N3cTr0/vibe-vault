---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\LogText.cs
---

# PartnerTool\LogText.cs

```csharp
using System.Globalization;
using System.Text;

namespace PartnerTool;

/// <summary>
/// Reduces log text to plain 7-bit ASCII so shared logs never "mojibake" in a ticketing / RMM tool
/// that misreads UTF-8 (e.g. "Â·", "â", "âââ", the "ï»¿" BOM). Applied ONLY at the file-write
/// sinks — the live in-app log and status lines keep their nicer glyphs (▶ ● — · …). Decorative
/// glyphs map to ASCII tokens; accented letters fold to their base letter (é → e); anything else
/// non-ASCII is dropped.
/// </summary>
public static class LogText
{
    /// <summary>UTF-8 with NO BOM — pure-ASCII content is then byte-identical under any code page.</summary>
    public static readonly Encoding Utf8NoBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);

    private static readonly (string From, string To)[] Glyphs =
    {
        ("▶", ">"),  ("●", "*"),   ("■", "*"),   ("◆", "*"),
        ("━", "-"),  ("─", "-"),   ("–", "-"),   ("—", "-"),
        ("»", ">"),  ("«", "<"),   ("•", "-"),   ("·", "-"),
        ("✓", "[OK]"), ("✔", "[OK]"), ("✗", "[X]"), ("⚠", "[!]"),
        ("→", "->"), ("←", "<-"),  ("…", "..."),
        ("“", "\""), ("”", "\""),  ("‘", "'"),   ("’", "'"),
        ("®", "(R)"), ("™", "(TM)"), ("©", "(C)"), ("°", " deg"), ("±", "+/-"),
    };

    public static string ToAscii(string? s)
    {
        if (string.IsNullOrEmpty(s)) return s ?? "";

        foreach (var (from, to) in Glyphs) s = s.Replace(from, to);

        // Fold accents (é → e, ü → u) by decomposing then dropping the combining marks.
        var norm = s.Normalize(NormalizationForm.FormD);
        var sb = new StringBuilder(norm.Length);
        foreach (var ch in norm)
        {
            if (CharUnicodeInfo.GetUnicodeCategory(ch) == UnicodeCategory.NonSpacingMark) continue;
            // Keep tab/newlines and the printable ASCII range; drop everything else non-ASCII.
            if (ch is '\r' or '\n' or '\t' || (ch >= ' ' && ch <= '~')) sb.Append(ch);
        }
        return sb.ToString();
    }
}
```
