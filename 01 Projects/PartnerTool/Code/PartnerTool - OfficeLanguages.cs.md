---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\OfficeLanguages.cs
---

# PartnerTool\OfficeLanguages.cs

```csharp
using System.Globalization;
using System.Text.RegularExpressions;
using Microsoft.Win32;

namespace PartnerTool;

/// <summary>One installed Click-to-Run Office language, as listed in Add/Remove Programs.</summary>
public record OfficeLanguagePack(string Culture, string DisplayName, string FriendlyName, string UninstallString)
{
    public bool IsEnglish => Culture.StartsWith("en", StringComparison.OrdinalIgnoreCase);
}

public class OfficeLanguageScan
{
    public bool OfficeFound;
    public List<OfficeLanguagePack> Packs { get; } = new();
    public List<OfficeLanguagePack> English    => Packs.Where(p => p.IsEnglish).ToList();
    public List<OfficeLanguagePack> NonEnglish => Packs.Where(p => !p.IsEnglish).ToList();
}

/// <summary>
/// Detects and removes Microsoft 365 / Office (Click-to-Run) language packs. Each installed
/// language registers its own Add/Remove Programs entry whose UninstallString is the C2R ARP
/// remover with a <c>culture=xx-xx</c> token, e.g.:
///   "…\ClickToRun\OfficeClickToRun.exe" scenario=install scenariosubtype=ARP sourcetype=None
///     productstoremove=O365ProPlusRetail.16_fr-fr_x-none culture=fr-fr version.16=16.0
/// Display names are localized (e.g. the Portuguese pack reads "…para Pequenas e Médias Empresas"),
/// so we key off the culture code, never the display text. Removing a language = running its own
/// uninstall command (exactly what Windows runs from Programs &amp; Features) with silent flags.
/// </summary>
public static class OfficeLanguages
{
    private static readonly Regex CultureRx =
        new(@"culture=([a-zA-Z]{2,3}(?:-[a-zA-Z0-9]{2,8})?)", RegexOptions.IgnoreCase | RegexOptions.Compiled);

    private static readonly string[] UninstallRoots =
    {
        @"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
        @"SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
    };

    public static OfficeLanguageScan Scan()
    {
        var scan = new OfficeLanguageScan();
        foreach (var root in UninstallRoots)
        {
            using var key = Registry.LocalMachine.OpenSubKey(root);
            if (key == null) continue;

            foreach (var sub in key.GetSubKeyNames())
            {
                using var entry = key.OpenSubKey(sub);
                if (entry?.GetValue("UninstallString") is not string uninstall || uninstall.Length == 0)
                    continue;

                // Only Click-to-Run per-language ARP entries.
                if (uninstall.IndexOf("OfficeClickToRun.exe", StringComparison.OrdinalIgnoreCase) < 0) continue;
                if (uninstall.IndexOf("productstoremove=", StringComparison.OrdinalIgnoreCase) < 0) continue;

                scan.OfficeFound = true;

                var m = CultureRx.Match(uninstall);
                if (!m.Success) continue;
                var culture = m.Groups[1].Value;
                if (culture.Equals("x-none", StringComparison.OrdinalIgnoreCase)) continue;

                // 32- and 64-bit views can both list the same culture — keep one.
                if (scan.Packs.Any(p => p.Culture.Equals(culture, StringComparison.OrdinalIgnoreCase))) continue;

                var display = entry.GetValue("DisplayName") as string ?? sub;
                string friendly;
                try { friendly = CultureInfo.GetCultureInfo(culture).DisplayName; }
                catch { friendly = culture; }

                scan.Packs.Add(new OfficeLanguagePack(culture, display, friendly, uninstall));
            }
        }
        return scan;
    }

    /// <summary>
    /// Split an ARP uninstall string into exe + args and append silent flags so the removal
    /// runs without UI and closes any open Office apps instead of blocking.
    /// </summary>
    public static (string exe, string args) BuildSilentRemoval(string uninstallString)
    {
        var s = uninstallString.Trim();
        string exe, args;
        if (s.StartsWith('"'))
        {
            int end = s.IndexOf('"', 1);
            exe  = s.Substring(1, end - 1);
            args = s[(end + 1)..].Trim();
        }
        else
        {
            int sp = s.IndexOf(' ');
            exe  = sp < 0 ? s : s[..sp];
            args = sp < 0 ? "" : s[(sp + 1)..].Trim();
        }

        if (args.IndexOf("DisplayLevel=", StringComparison.OrdinalIgnoreCase) < 0)
            args += " DisplayLevel=False";
        if (args.IndexOf("forceappshutdown=", StringComparison.OrdinalIgnoreCase) < 0)
            args += " forceappshutdown=True";
        return (exe, args);
    }
}
```
