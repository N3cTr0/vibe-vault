---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SoftwarePage.xaml.cs
---

# PartnerTool\Pages\SoftwarePage.xaml.cs

```csharp
using System.IO;
using System.Text.Json;
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class SoftwarePage : UserControl
{
    private List<AppEntry> _all = [];
    private bool _busy;
    private bool _checking;
    private bool _detectedOnce;

    public SoftwarePage()
    {
        InitializeComponent();
        Loaded += async (_, _) => await EnsureLoadedAsync();
    }

    /// <summary>
    /// Load startup programs + installed software and run the slow winget "already installed"
    /// detection. Runs once (guarded); can be pre-cached in the background from startup so the
    /// list is ready the moment the tech opens the tab.
    /// </summary>
    public async Task EnsureLoadedAsync()
    {
        if (_detectedOnce) return;   // first visit only — don't re-scan when tabbing away and back
        _detectedOnce = true;

        InstallLoadingOverlay.Visibility = Visibility.Visible;
        BtnInstall.IsEnabled = false;
        try
        {
            // Startup is a quick registry read — show it first, before the slow winget detection.
            ShowStartup(await Task.Run(StartupInfo.Collect));   // startup programs (merged from Inventory)
            await LoadAsync();                  // installed-software list (registry — quick)
            await RefreshInstalledStateAsync(); // detect + lock already-installed apps (slow: winget export)
        }
        finally
        {
            InstallLoadingOverlay.Visibility = Visibility.Collapsed;
            BtnInstall.IsEnabled = true;
        }
    }

    // ── Startup programs (merged from the old Inventory tab) ──────────────
    private void ShowStartup(List<StartupEntry> startup)
    {
        var sorted = startup
            .OrderByDescending(s => s.Enabled)                     // enabled first, disabled at the bottom
            .ThenBy(s => s.Name, StringComparer.OrdinalIgnoreCase)
            .ToList();
        IcStartup.ItemsSource   = sorted;
        TxtNoStartup.Visibility = sorted.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
    }

    private async void StartupRefresh_Click(object sender, RoutedEventArgs e)
        => ShowStartup(await Task.Run(StartupInfo.Collect));

    private async void ToggleStartup_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: StartupEntry entry }) return;

        // Never let the tool disable a security product's startup entry (AV/EDR trays & agents).
        if (entry.Enabled && StartupInfo.IsSecurityCritical(entry))
        {
            MessageWindow.Show("Startup", $"“{entry.Name}” is security software",
                "Disabling a security product's startup entry would weaken this machine's protection, " +
                "so it's blocked here.", MessageKind.Warning, Window.GetWindow(this));
            return;
        }

        ActivityLog.Action("Startup", $"{(entry.Enabled ? "Disable" : "Enable")} startup item “{entry.Name}” ({entry.LocationText})");
        try { StartupInfo.SetEnabled(entry, !entry.Enabled); }
        catch (Exception ex)
        {
            MessageWindow.Show("Startup", "Couldn't change this item",
                ex.Message, MessageKind.Error, Window.GetWindow(this));
            return;
        }
        ShowStartup(await Task.Run(StartupInfo.Collect));   // re-read + re-sort
    }

    // ── Detect already-installed apps and lock their checkboxes ───────────

    /// <summary>
    /// Ask winget which of the listed apps are already installed, then tick and
    /// disable those checkboxes so they can't be reinstalled.
    /// </summary>
    private async Task RefreshInstalledStateAsync()
    {
        if (_checking) return;
        _checking = true;
        BtnInstall.IsEnabled = false;
        try
        {
            // Phase 1 — instant: match the registry list we already have. Catches apps
            // installed outside winget (e.g. a Zoom downloaded from the web) that winget
            // can't correlate to a catalog ID.
            ApplyInstalled(_ => false);
            TxtInstallStatus.Text = "Checking which apps are already installed…";

            // Phase 2 — add winget's exact-ID matches (differently-named apps like the
            // Adobe Reader or the .NET runtimes that the registry name won't catch).
            var wingetIds = await DetectInstalledIdsAsync();
            int locked = ApplyInstalled(wingetIds.Contains);

            TxtInstallStatus.Text = locked == 0
                ? "None of these are installed yet."
                : $"{locked} already installed on this PC (ticked & locked).";
        }
        finally
        {
            _checking = false;
            if (!_busy) BtnInstall.IsEnabled = true;
        }
    }

    /// <summary>
    /// Tick + lock every app that's installed — either matched by winget ID
    /// (<paramref name="wingetHasId"/>) or found in the registry by name. Returns the
    /// number locked.
    /// </summary>
    private int ApplyInstalled(Func<string, bool> wingetHasId)
    {
        int locked = 0;
        foreach (var cb in AllCheckBoxes(PnlInstall))
        {
            var id   = cb.Tag?.ToString() ?? "";
            var name = cb.Content?.ToString() ?? "";
            bool isInstalled = (id.Length > 0 && wingetHasId(id)) || RegistryHasApp(name);

            if (isInstalled)
            {
                cb.IsChecked = true;
                cb.IsEnabled = false;
                cb.ToolTip   = "Already installed";
                locked++;
            }
            else
            {
                cb.IsEnabled = true;
                cb.ToolTip   = null;
            }
        }
        return locked;
    }

    /// <summary>True if an installed program matches this display name — exactly, or as
    /// the leading part of the registry name (e.g. "VLC" ⇒ "VLC media player").</summary>
    private bool RegistryHasApp(string displayName)
    {
        if (string.IsNullOrWhiteSpace(displayName)) return false;
        foreach (var a in _all)
            if (a.Name.Equals(displayName, StringComparison.OrdinalIgnoreCase) ||
                a.Name.StartsWith(displayName + " ", StringComparison.OrdinalIgnoreCase))
                return true;
        return false;
    }

    /// <summary>Run <c>winget export</c> and return the set of installed package IDs.</summary>
    private static async Task<HashSet<string>> DetectInstalledIdsAsync()
    {
        var set = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        if (!WingetLocator.IsAvailable()) return set;   // can't detect what's installed without it

        var outFile = Path.Combine(Path.GetTempPath(), $"pt_winget_{Guid.NewGuid():N}.json");
        try
        {
            await ProcessRunner.RunAsync(WingetLocator.Path(),
                $"export -o \"{outFile}\" --accept-source-agreements --disable-interactivity",
                System.Text.Encoding.UTF8, _ => { });

            if (File.Exists(outFile))
            {
                using var doc = JsonDocument.Parse(await File.ReadAllTextAsync(outFile));
                if (doc.RootElement.TryGetProperty("Sources", out var sources))
                    foreach (var src in sources.EnumerateArray())
                        if (src.TryGetProperty("Packages", out var pkgs))
                            foreach (var pkg in pkgs.EnumerateArray())
                                if (pkg.TryGetProperty("PackageIdentifier", out var idEl) &&
                                    idEl.GetString() is { Length: > 0 } id)
                                    set.Add(id);
            }
        }
        catch { }
        finally { try { File.Delete(outFile); } catch { } }
        return set;
    }

    // ── Installed software list ───────────────────────────────────────────

    private async Task LoadAsync()
    {
        TxtCount.Text = "Loading…";
        _all = await Task.Run(GetInstalledApps);
        ApplyFilter(SearchBox.Text);
    }

    private void Search_Changed(object sender, TextChangedEventArgs e)
        => ApplyFilter(SearchBox.Text);

    private void ApplyFilter(string query)
    {
        var filtered = string.IsNullOrWhiteSpace(query)
            ? _all
            : _all.Where(a => a.Name.Contains(query, StringComparison.OrdinalIgnoreCase)
                            || a.Publisher.Contains(query, StringComparison.OrdinalIgnoreCase)).ToList();

        SoftwareList.ItemsSource = filtered;
        TxtCount.Text = filtered.Count == _all.Count
            ? $"{_all.Count} apps"
            : $"{filtered.Count} of {_all.Count}";
    }

    private void UninstallApp_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: AppEntry app } || string.IsNullOrWhiteSpace(app.Uninstall)) return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;

        // Security: this tool runs elevated. A per-user (HKCU) uninstall string is writable by a
        // standard user, so running it from here would let them execute a command as admin. Refuse
        // to launch user-controlled uninstallers from the elevated process.
        if (!app.Machine)
        {
            MessageWindow.Show("Uninstall", "Per-user app — not removed from here",
                $"“{app.Name}” is registered under the current user (HKCU), so its uninstall command is " +
                "user-editable. Running it from this elevated tool could let a standard user run code as " +
                "administrator, so it's blocked.\n\nUninstall it from Windows Settings ▸ Apps in the user's " +
                "own session instead.", MessageKind.Warning, Window.GetWindow(this));
            return;
        }

        if (!MessageWindow.Confirm("Uninstall", $"Uninstall {app.Name}?",
                "This launches the program's own uninstaller. Follow its prompts to finish, then hit " +
                "Refresh (clear the search box) to update the list.", MessageKind.Warning, Window.GetWindow(this)))
            return;
        ActivityLog.Action("Software", $"Uninstall {app.Name} — {app.Uninstall}");
        try
        {
            // Uninstall strings are command lines: `"C:\..\unins.exe" /S` or `MsiExec.exe /X{GUID}`.
            var cmd = app.Uninstall.Trim();
            System.Diagnostics.ProcessStartInfo psi;
            if (cmd.StartsWith('"'))
            {
                int end  = cmd.IndexOf('"', 1);
                var exe  = cmd.Substring(1, end - 1);
                var args = cmd[(end + 1)..].Trim();
                psi = new System.Diagnostics.ProcessStartInfo(exe, args) { UseShellExecute = true };
            }
            else
            {
                int sp = cmd.IndexOf(' ');
                psi = sp < 0
                    ? new System.Diagnostics.ProcessStartInfo(cmd) { UseShellExecute = true }
                    : new System.Diagnostics.ProcessStartInfo(cmd[..sp], cmd[(sp + 1)..]) { UseShellExecute = true };
            }
            System.Diagnostics.Process.Start(psi);
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Uninstall", "Couldn't start the uninstaller",
                ex.Message, MessageKind.Error, Window.GetWindow(this));
        }
    }

    // Maps the shared inventory to the page's bindable AppEntry (adds the install/uninstall flags).
    private static List<AppEntry> GetInstalledApps()
        => SoftwareInventory.Collect().Select(a => new AppEntry
        {
            Name        = a.Name,
            Version     = a.Version,
            Publisher   = a.Publisher,
            InstallDate = a.InstallDate,
            Uninstall   = a.Uninstall,   // HKCU ones are user-writable — see UninstallApp_Click.
            Machine     = a.Machine,
        }).ToList();

    // ── Installer ─────────────────────────────────────────────────────────

    private void Log(string line)
        => Dispatcher.Invoke(() =>
        {
            TxtLog.Text += line + "\n";
            if (ChkAutoScroll.IsChecked == true) LogScroll.ScrollToBottom();
        });

    private async void InstallSelected_Click(object sender, RoutedEventArgs e)
    {
        if (_busy) return;

        // Skip anything already installed (ticked + disabled) — only take live ticks.
        var apps = AllCheckBoxes(PnlInstall)
            .Where(c => c.IsChecked == true && c.IsEnabled)
            .Select(c => (Name: c.Content?.ToString() ?? "", Id: c.Tag?.ToString() ?? ""))
            .Where(a => a.Id.Length > 0)
            .ToList();

        PnlLog.Visibility = Visibility.Visible;
        if (apps.Count == 0) { Log("No apps selected."); return; }

        _busy = true;
        BtnInstall.IsEnabled = false;
        BtnInstall.Content   = "Installing…";
        TxtLog.Text          = "";

        var winget = WingetLocator.Path();
        if (winget.Length == 0)
        {
            Log("✗ " + WingetLocator.Unavailable);
            _busy = false;
            BtnInstall.IsEnabled = true;
            BtnInstall.Content   = "Install Selected";
            return;
        }

        try
        {
            foreach (var (name, id) in apps)
            {
                Log($"▶ Installing {name}");
                ActivityLog.Action("Software", $"Install {name} ({id})");
                var code = await ProcessRunner.RunAsync(winget,
                    $"install --id {id} -e --silent --accept-source-agreements --accept-package-agreements --disable-interactivity",
                    System.Text.Encoding.UTF8, l => Log($"  {l}"));
                Log(code == 0 ? "  ✓ done" : $"  finished (exit {code})");
            }
            Log("━━━ Install complete ━━━");
        }
        catch (Exception ex) { Log($"  Error: {ex.Message}"); }
        finally
        {
            _busy = false;
            BtnInstall.IsEnabled = true;
            BtnInstall.Content   = "Install Selected";
        }

        // Re-scan registry + winget so anything we just installed flips to ticked & locked.
        await LoadAsync();
        await RefreshInstalledStateAsync();
    }

    private static IEnumerable<CheckBox> AllCheckBoxes(DependencyObject root)
    {
        foreach (var child in LogicalTreeHelper.GetChildren(root).OfType<DependencyObject>())
        {
            if (child is CheckBox cb) yield return cb;
            foreach (var nested in AllCheckBoxes(child)) yield return nested;
        }
    }
}

public class AppEntry
{
    public string Name        { get; set; } = "";
    public string Version     { get; set; } = "";
    public string Publisher   { get; set; } = "";
    public string InstallDate { get; set; } = "";
    public string Uninstall   { get; set; } = "";   // registry (Quiet)UninstallString
    public bool   Machine     { get; set; }          // true = HKLM (admin-writable), false = HKCU

    public bool CanUninstall => !string.IsNullOrWhiteSpace(Uninstall);
}
```
