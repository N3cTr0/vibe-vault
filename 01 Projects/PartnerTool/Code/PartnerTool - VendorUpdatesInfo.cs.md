---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\VendorUpdatesInfo.cs
---

# PartnerTool\VendorUpdatesInfo.cs

```csharp
using System.IO;
using System.Management;
using System.Text.RegularExpressions;
using System.Xml.Linq;

namespace PartnerTool;

public record VendorUpdate(string Title, string Detail);

/// <summary>
/// Result of a vendor (OEM) driver/BIOS/firmware update scan: a human-readable status line
/// plus the list of applicable updates the OEM tool reported.
/// </summary>
public class VendorScanResult
{
    public string             Status  { get; set; } = "";
    public List<VendorUpdate> Updates { get; set; } = new();
}

/// <summary>
/// Scan-only check for OEM driver/BIOS/firmware updates using the detected manufacturer's
/// command-line tooling — Dell Command Update (dcu-cli), Lenovo (LSUClient), HP (HPIA).
/// If the tool isn't present we install/download it first (Dell via winget, Lenovo's LSUClient
/// is a PowerShell module that self-installs, HP Image Assistant is fetched from HP), then scan.
/// These talk to the vendor and can take a couple of minutes, so only ever run on demand.
/// The scan itself installs no updates; it just reports what's available. Every path is guarded
/// so a missing tool or a malformed report degrades to a status message, never an exception.
///
/// This is the scan-only adaptation of the production NinjaOne hardware update script
/// (reference/HUS_MultiVendorUpdate_v1.14.1.ps1) — same vendor detection, tool auto-install and
/// CLI invocations, but it lists instead of installing. Keep the two in sync.
/// </summary>
public static class VendorUpdatesInfo
{
    /// <param name="installIfMissing">
    /// When true (manual scan) the OEM tool is installed/downloaded if absent. When false
    /// (auto-scan on tab open) a missing tool is just reported — we never install on open.
    /// </param>
    public static async Task<VendorScanResult> ScanAsync(Action<string>? onStatus = null, bool installIfMissing = true)
    {
        var r = new VendorScanResult();

        string maker, model;
        try { (maker, model) = await Task.Run(GetMakerModel); }
        catch { maker = ""; model = ""; }

        var tool = ManufacturerTools.Get(maker, model);
        if (tool == null)
        {
            r.Status = ManufacturerTools.IsVirtualMachine(maker, model)
                ? "Virtual machine — no OEM driver/BIOS updates (use Windows Update)."
                : string.IsNullOrWhiteSpace(maker)
                    ? "Could not detect the hardware manufacturer."
                    : $"No command-line update scanner is available for \"{maker}\".";
            return r;
        }

        var name = tool.DisplayName.ToLowerInvariant();
        try
        {
            if (name.Contains("dell"))                           return await ScanDellAsync(r, onStatus, installIfMissing);
            if (name.Contains("lenovo"))                         return await ScanLenovoAsync(r, onStatus, installIfMissing);
            if (name.Contains("hp") || name.Contains("hewlett")) return await ScanHpAsync(r, model, onStatus, installIfMissing);

            r.Status = $"{tool.DisplayName} has no command-line scanner — open it from the update " +
                       "sources below to check for driver/firmware updates.";
        }
        catch (Exception ex) { r.Status = $"Vendor scan failed: {ex.Message}"; }
        return r;
    }

    // ── Dell — dcu-cli.exe /scan -report=<dir>  →  DCUApplicableUpdates.xml ──
    private static async Task<VendorScanResult> ScanDellAsync(VendorScanResult r, Action<string>? onStatus, bool installIfMissing)
    {
        var cli = FindDellCli();
        if (cli == null)
        {
            if (!installIfMissing)
            {
                r.Status = "Dell Command Update isn't installed (open the Dell update source below to add it).";
                return r;
            }
            onStatus?.Invoke("Dell Command Update isn't installed — installing it via winget (one-time, can take a few minutes)…");
            await InstallViaWingetAsync("Dell.CommandUpdate.Universal");
            cli = FindDellCli();
        }
        if (cli == null)
        {
            r.Status = "Couldn't install Dell Command Update automatically (winget may be unavailable). " +
                       "Install it from the Dell update source below, then scan again.";
            return r;
        }

        onStatus?.Invoke("Scanning with Dell Command Update… this can take a few minutes.");

        var dir = Path.Combine(Path.GetTempPath(), "pci_dcu_scan");
        var xml = Path.Combine(dir, "DCUApplicableUpdates.xml");
        try { Directory.CreateDirectory(dir); } catch { }
        try { if (File.Exists(xml)) File.Delete(xml); } catch { }   // don't read a stale report

        await ProcessRunner.RunCaptureAsync(cli, $"/scan -report=\"{dir}\"", 300000);

        if (File.Exists(xml))
        {
            try
            {
                var doc = XDocument.Load(xml);
                foreach (var u in doc.Descendants().Where(e => e.Name.LocalName == "update"))
                {
                    var title = Field(u, "name");
                    if (string.IsNullOrWhiteSpace(title)) continue;
                    r.Updates.Add(new VendorUpdate(title, Join(Field(u, "type", "category"), Field(u, "version"))));
                }
            }
            catch { }
        }

        r.Status = r.Updates.Count == 0
            ? "Dell Command Update: system is up to date."
            : $"Dell Command Update: {r.Updates.Count} driver/firmware update(s) available.";
        return r;
    }

    private static string? FindDellCli() => new[]
    {
        @"C:\Program Files\Dell\CommandUpdate\dcu-cli.exe",
        @"C:\Program Files (x86)\Dell\CommandUpdate\dcu-cli.exe",
    }.FirstOrDefault(File.Exists);

    // ── Lenovo — LSUClient PowerShell module (self-installs if missing) ──
    private static async Task<VendorScanResult> ScanLenovoAsync(VendorScanResult r, Action<string>? onStatus, bool installIfMissing)
    {
        onStatus?.Invoke("Scanning with Lenovo LSUClient…");

        // Install the module on a manual scan; on auto-scan just bail out if it isn't already there.
        var moduleStep = installIfMissing ? @"
if (-not (Get-Module -Name LSUClient -ListAvailable)) {
    if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue | Where-Object { $_.Version -ge '2.8.5.201' })) {
        Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope CurrentUser | Out-Null
    }
    Install-Module -Name LSUClient -Force -Scope CurrentUser -AllowClobber
}" : @"
if (-not (Get-Module -Name LSUClient -ListAvailable)) { Write-Output 'NOMODULE'; exit }";

        var ps = @"
$ErrorActionPreference = 'SilentlyContinue'
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12" + moduleStep + @"
Import-Module LSUClient -Force
$u = Get-LSUpdate | Where-Object { $_.Installer.Unattended }
foreach ($x in $u) { Write-Output (""UPD|"" + $x.Title + ""|"" + $x.Category + ""|"" + $x.Version) }
Write-Output 'DONE'
";
        var script = Path.Combine(Path.GetTempPath(), "pci_lsu_scan.ps1");
        try
        {
            await File.WriteAllTextAsync(script, ps);
            var outText = await ProcessRunner.RunCaptureAsync("powershell.exe",
                $"-ExecutionPolicy Bypass -NonInteractive -File \"{script}\"", 300000);

            if (outText.Contains("NOMODULE", StringComparison.Ordinal))
            {
                r.Status = "Lenovo update scanner (LSUClient) isn't installed yet.";
                return r;
            }

            foreach (var line in outText.Replace("\r", "").Split('\n'))
            {
                if (!line.StartsWith("UPD|", StringComparison.Ordinal)) continue;
                var parts = line.Split('|');
                var title = parts.Length > 1 ? parts[1].Trim() : "";
                if (string.IsNullOrWhiteSpace(title)) continue;
                var cat = parts.Length > 2 ? parts[2].Trim() : "";
                var ver = parts.Length > 3 ? parts[3].Trim() : "";
                r.Updates.Add(new VendorUpdate(title, Join(cat, ver)));
            }
        }
        finally { try { File.Delete(script); } catch { } }

        r.Status = r.Updates.Count == 0
            ? "Lenovo (LSUClient): no applicable updates found."
            : $"Lenovo: {r.Updates.Count} driver/firmware update(s) available.";
        return r;
    }

    // ── HP — HP Image Assistant analyze (downloaded from HP if missing) ──
    // HPIA only supports HP's *commercial* line (EliteBook/ProBook/ZBook, Pro/Elite desktops).
    // Consumer models (Pavilion/Envy/Spectre/OmniBook) aren't in HP's reference database — HPIA
    // fails with 16386 — so skip them up front and point the tech at Windows Update instead.
    private static async Task<VendorScanResult> ScanHpAsync(VendorScanResult r, string model, Action<string>? onStatus, bool installIfMissing)
    {
        if (!IsCommercialHp(model))
        {
            r.Status = $"HP \"{model}\" looks like a consumer model — HP Image Assistant only supports " +
                       "commercial HP models (EliteBook/ProBook/ZBook, HP Pro/Elite desktops). " +
                       "Use Windows Update for driver updates on this device.";
            return r;
        }

        // Lock the folder down first: we're about to download an .exe here and then run it as
        // admin, so it must not be writable (or plantable) by a standard user.
        SecureDirectory.EnsureHardened(@"C:\PCI\Tools");

        const string hpia = @"C:\PCI\Tools\HPImageAssistant.exe";
        if (!File.Exists(hpia))
        {
            if (!installIfMissing)
            {
                r.Status = "HP Image Assistant isn't installed yet (it downloads on a manual vendor scan).";
                return r;
            }
            onStatus?.Invoke("HP Image Assistant isn't present — downloading it from HP (one-time)…");
            await EnsureHpiaAsync();
        }
        if (!File.Exists(hpia))
        {
            r.Status = "Couldn't download HP Image Assistant automatically. Run HP Support Assistant " +
                       "to check for updates, or try again.";
            return r;
        }

        onStatus?.Invoke("Scanning with HP Image Assistant… this can take a few minutes.");

        var dir = Path.Combine(Path.GetTempPath(), "pci_hpia_scan");
        try { Directory.CreateDirectory(dir); } catch { }

        await ProcessRunner.RunCaptureAsync(hpia,
            $"/Operation:Analyze /Action:List /Silent /Category:All /ReportFolder:\"{dir}\" /SoftpaqDownloadFolder:\"{dir}\"",
            300000);

        try
        {
            var report = new DirectoryInfo(dir).GetFiles("*.xml")
                .OrderByDescending(f => f.LastWriteTimeUtc).FirstOrDefault();
            if (report != null)
            {
                var doc = XDocument.Load(report.FullName);
                foreach (var rec in doc.Descendants().Where(e => e.Name.LocalName == "Recommendation"))
                {
                    var title = Field(rec, "TargetComponent", "Name");
                    if (string.IsNullOrWhiteSpace(title)) continue;
                    r.Updates.Add(new VendorUpdate(title, Join(Field(rec, "TargetCategory", "Category"),
                                                              Field(rec, "TargetVersion", "Version"))));
                }
            }
        }
        catch { }

        r.Status = r.Updates.Count == 0
            ? "HP Image Assistant: system is up to date (or no report produced)."
            : $"HP: {r.Updates.Count} driver/firmware update(s) available.";
        return r;
    }

    /// <summary>
    /// Download the latest HP Image Assistant from HP and silently extract it to C:\PCI\Tools.
    /// Mirrors the production HUS script: scrape HP's HPIA page for the current hp-hpia-*.exe
    /// SoftPaq, then run the self-extractor with "/s /f &lt;folder&gt; /e" (arg order matters).
    /// Public so Update All can also fetch it when missing (like the HUS script does).
    /// </summary>
    public static async Task EnsureHpiaAsync()
    {
        const string ps = @"
$ErrorActionPreference = 'SilentlyContinue'
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
New-Item -ItemType Directory -Force 'C:\PCI\Tools' | Out-Null
$page = Invoke-WebRequest 'https://ftp.hp.com/pub/caps-softpaq/cmit/HPIA.html' -UseBasicParsing
$link = ($page.Links | Where-Object { $_.href -match 'hp-hpia-' } | Select-Object -First 1).href
if (-not $link) { exit 1 }
$exe = Join-Path $env:TEMP 'pci_hpia_setup.exe'
Invoke-WebRequest $link -OutFile $exe -UseBasicParsing
# The download is a SoftPaq self-extractor: /s silent, /f <folder> target, /e extract
Start-Process -FilePath $exe -ArgumentList '/s /f ""C:\PCI\Tools"" /e' -Wait -NoNewWindow
Remove-Item $exe -Force -ErrorAction SilentlyContinue
";
        var script = Path.Combine(Path.GetTempPath(), "pci_hpia_get.ps1");
        try
        {
            await File.WriteAllTextAsync(script, ps);
            await ProcessRunner.RunCaptureAsync("powershell.exe",
                $"-ExecutionPolicy Bypass -NonInteractive -File \"{script}\"", 600000);
        }
        finally { try { File.Delete(script); } catch { } }
    }

    // ── helpers ──
    /// <summary>Install an OEM tool via winget (also used by Update All when the tool is missing,
    /// mirroring the HUS script's auto-install-prerequisites behavior).</summary>
    public static async Task InstallViaWingetAsync(string id)
    {
        if (!WingetLocator.IsAvailable()) return;

        await ProcessRunner.RunCaptureAsync(WingetLocator.Path(),
            $"install --id {id} --silent --accept-source-agreements --accept-package-agreements --disable-interactivity",
            600000);
    }


    /// <summary>Manufacturer + model from Win32_ComputerSystem (one WMI read), best-effort.</summary>
    private static (string Maker, string Model) GetMakerModel()
    {
        string maker = "", model = "";
        try
        {
            using var s = new ManagementObjectSearcher("SELECT Manufacturer, Model FROM Win32_ComputerSystem");
            foreach (ManagementObject o in s.Get())
            {
                maker = (o["Manufacturer"] as string ?? "").Trim();
                model = (o["Model"] as string ?? "").Trim();
                break;
            }
        }
        catch { }
        return (maker, model);
    }

    private static readonly Regex HpCommercial = new(
        @"EliteBook|ProBook|ZBook|EliteDesk|ProDesk|ProOne|EliteOne|Z[0-9]|HP\s+Pro|HP\s+Elite",
        RegexOptions.IgnoreCase | RegexOptions.Compiled);

    private static bool IsCommercialHp(string model) =>
        !string.IsNullOrWhiteSpace(model) && HpCommercial.IsMatch(model);

    private static string Field(XElement el, params string[] names)
    {
        foreach (var n in names)
        {
            var child = el.Elements().FirstOrDefault(e => string.Equals(e.Name.LocalName, n, StringComparison.OrdinalIgnoreCase));
            if (child != null && !string.IsNullOrWhiteSpace(child.Value)) return child.Value.Trim();
            var attr = el.Attributes().FirstOrDefault(a => string.Equals(a.Name.LocalName, n, StringComparison.OrdinalIgnoreCase));
            if (attr != null && !string.IsNullOrWhiteSpace(attr.Value)) return attr.Value.Trim();
        }
        return "";
    }

    private static string Join(params string[] bits) =>
        string.Join("  ·  ", bits.Where(s => !string.IsNullOrWhiteSpace(s)));
}
```
