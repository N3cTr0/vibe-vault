---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\Sensitive.cs
---

# src\Octavia.Core\Core\Sensitive.cs

```csharp
using System.Text.RegularExpressions;

namespace Octavia.Core;

/// Whether the *name* of a setting says it holds a secret.
///
/// Lifted out of the diagnostics bundle in v0.48.0, when a second place needed the same
/// answer: the bundle redacts these before it leaves the building, and the settings window
/// must never draw one on screen. Two implementations would have been two opinions, and the
/// disagreement would only ever show up as a secret displayed by one of them.
internal static partial class Sensitive
{
    private static readonly string[] Words = ["key", "token", "secret", "password", "credential"];

    /// **A plain substring match blanked `Hotkey` and `MaxTokens`**, which are two of the most
    /// useful lines in a config file. Match whole words of the name instead — splitting on
    /// camel case and on punctuation, so `UNIFI_API_KEY`, `apiKey` and `ApiKey` all resolve to
    /// the same three words — and take the plural off each.
    ///
    /// Over-redaction is not the safe default here: it quietly destroys the thing a bundle
    /// exists to carry, and greys out settings a person came to the window to read.
    public static bool Looks(string name) =>
        Boundaries().Split(name)
                    .Where(word => word.Length > 0)
                    .Select(word => word.TrimEnd('s', 'S'))
                    .Any(word => Words.Contains(word, StringComparer.OrdinalIgnoreCase));

    /* Three boundaries, and the middle one is the fix.

       The original rule split before *every* capital — `(?<!^)(?=[A-Z])` — which reads
       correctly for `apiKey` and falls apart on a name that is entirely capitals.
       **`UNIFI_API_KEY` split into thirteen single letters**, none of which is "key", so the
       one name this was written to catch was the one it missed. It had been wrong the whole
       time and nothing noticed, because the only caller was a redactor whose failure is
       invisible by definition — you do not see the secret it *should* have removed.

       Splitting only at a *transition into* a capital fixes it: after a lower-case letter or
       a digit, or where an acronym runs into a word (`APIKey` -> `API`, `Key`). A run of
       capitals stays one word, and `UNIFI_API_KEY` becomes `UNIFI`, `API`, `KEY`. */
    [GeneratedRegex(@"(?<=[a-z0-9])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])|[^A-Za-z0-9]+")]
    private static partial Regex Boundaries();
}
```
