---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\UpdatesPage.xaml.cs
---

# PartnerTool\Pages\UpdatesPage.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Text.RegularExpressions;
using System.Windows;
using System.Windows.Controls;
using AppInstallState = Windows.ApplicationModel.Store.Preview.InstallControl.AppInstallState;

namespace PartnerTool.Pages;

public partial class UpdatesPage : UserControl
{
    private string?           _mfrLaunchTarget;
    private ManufacturerTool? _mfrTool;
    private string?           _mfrExe;
    private bool              _loadStarted;
    private bool              _historyLoaded;
    private List<UpdateEntry> _allHistory = new();

    public UpdatesPage()
    {
        InitializeComponent();
        Loaded += async (_, _) => await EnsureLoadedAsync();
    }

    /// <summary>
    /// Detect the OEM tool, load Windows Update history, and scan all update sources — runs once,
    /// whether triggered by the page being shown or pre-cached in the background from startup so the
    /// data is already there when the tech opens the tab. The guard is set before any await so a
    /// navigation landing here mid-preload can't kick off a second scan.
    /// </summary>
    public async Task EnsureLoadedAsync()
    {
        if (_loadStarted) return;
        _loadStarted = true;
        await LoadMfrToolAsync();
        var history = LoadHistoryAsync();   // own guard — runs alongside the scans
        await ScanAllAsync();
        await history;
    }

    private async System.Threading.Tasks.Task LoadHistoryAsync()
    {
        if (_historyLoaded) return;     // WUA COM is slow — only the first time the page is shown
        _historyLoaded = true;
        try
        {
            var info = await System.Threading.Tasks.Task.Run(UpdateHistoryInfo.Collect);
            _allHistory = info.Recent;
            TxtLastPatched.Text = info.LastInstalled is { } lp
                ? $"Last update installed: {lp:dddd, d MMM yyyy}  ({(int)(DateTime.Now - lp).TotalDays} days ago)"
                : "Last update date unavailable.";
            IcHistory.ItemsSource     = info.Recent.Take(10).ToList();
            TxtNoHistory.Text         = "No update history available.";
            TxtNoHistory.Visibility   = info.Recent.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
            BtnMoreHistory.Visibility = info.Recent.Count > 10 ? Visibility.Visible : Visibility.Collapsed;
        }
        catch { TxtNoHistory.Text = "Update history unavailable."; }
    }

    private void MoreHistory_Click(object sender, RoutedEventArgs e)
    {
        var rows = _allHistory.Select(u => new ListRow(u.Title, u.Date.ToString("dddd, d MMM yyyy"), u.Result));
        new ListWindow("Windows Update History", rows) { Owner = Window.GetWindow(this) }.ShowDialog();
    }

    private async System.Threading.Tasks.Task LoadMfrToolAsync()
    {
        var (manufacturer, model) = await System.Threading.Tasks.Task.Run(SystemInfo.GetMakeModel);
        var tool = ManufacturerTools.Get(manufacturer, model);

        if (tool == null)
        {
            TxtMfrName.Text         = "Manufacturer Update Tool";
            TxtMfrDesc.Text         = ManufacturerTools.IsVirtualMachine(manufacturer, model)
                ? "This is a virtual machine — no manufacturer firmware/driver updates (use Windows Update)."
                : string.IsNullOrWhiteSpace(manufacturer)
                    ? "Could not detect hardware manufacturer."
                    : $"No known update tool for \"{manufacturer}\".";
            TxtMfrStatus.Text       = "● Not applicable";
            TxtMfrStatus.Foreground = StatusColors.Muted;
            BtnMfr.IsEnabled        = false;
            return;
        }

        _mfrTool        = tool;
        TxtMfrName.Text = tool.DisplayName;
        TxtMfrDesc.Text = tool.Description;

        var exe = await System.Threading.Tasks.Task.Run(tool.ResolveExe);
        _mfrExe = exe;

        if (exe != null)
        {
            _mfrLaunchTarget        = exe;
            TxtMfrStatus.Text       = "● Installed";
            TxtMfrStatus.Foreground = StatusColors.Green;
            BtnMfr.IsEnabled        = true;
        }
        else if (tool.StoreUri != null)
        {
            _mfrLaunchTarget        = tool.StoreUri;
            TxtMfrStatus.Text       = "● Available via Store";
            TxtMfrStatus.Foreground = StatusColors.Blue;
            BtnMfr.IsEnabled        = true;
        }
        else
        {
            TxtMfrStatus.Text       = "● Not installed";
            TxtMfrStatus.Foreground = StatusColors.Muted;
            BtnMfr.IsEnabled        = false;
        }
    }

    private void OpenMfr_Click(object sender, RoutedEventArgs e)
    {
        if (_mfrLaunchTarget != null) Launch(_mfrLaunchTarget);
    }

    private void OpenWindowsUpdate_Click(object sender, RoutedEventArgs e) =>
        Launch("ms-settings:windowsupdate");

    private void OpenStore_Click(object sender, RoutedEventArgs e) =>
        Launch("ms-windows-store:");

    private static void Launch(string target)
    {
        try { Process.Start(new ProcessStartInfo(target) { UseShellExecute = true }); }
        catch (Exception ex)
        {
            MessageWindow.Show("Error", "Couldn't open", ex.Message, MessageKind.Error);
        }
    }

    // ── AVAILABLE UPDATES — auto-scanned in parallel when the tab opens ───

    /// <summary>Run all scans together — they're network / subprocess bound, not CPU-heavy.</summary>
    private async Task ScanAllAsync()
        => await Task.WhenAll(ScanWindowsAsync(), ScanAppsAsync(), ScanVendorAsync(), ScanStoreAsync());

    private void SetStatus(TextBlock tb, string text, System.Windows.Media.Brush color)
    {
        tb.Visibility = Visibility.Visible;
        tb.Text       = text;
        tb.Foreground = color;
    }

    private async Task ScanWindowsAsync()
    {
        SetStatus(TxtNoPending, "● Scanning Windows Update…", StatusColors.Yellow);
        try
        {
            var updates = await Task.Run(PendingUpdatesInfo.Collect);
            IcPending.ItemsSource = updates;
            if (updates.Count == 0) SetStatus(TxtNoPending, "No pending Windows updates.", StatusColors.Muted);
            else TxtNoPending.Visibility = Visibility.Collapsed;
        }
        catch (Exception ex) { SetStatus(TxtNoPending, $"Scan failed: {ex.Message}", StatusColors.Red); }
    }

    private async Task ScanAppsAsync()
    {
        SetStatus(TxtNoOutdated, "● Scanning app updates (winget)…", StatusColors.Yellow);
        try
        {
            var apps = await OutdatedAppsInfo.CollectAsync();
            IcOutdated.ItemsSource = apps;
            if (apps.Count == 0) SetStatus(TxtNoOutdated, "All apps up to date (or winget unavailable).", StatusColors.Muted);
            else TxtNoOutdated.Visibility = Visibility.Collapsed;
        }
        catch (Exception ex) { SetStatus(TxtNoOutdated, $"Scan failed: {ex.Message}", StatusColors.Red); }
    }

    private async Task ScanStoreAsync()
    {
        SetStatus(TxtNoStore, "● Scanning Microsoft Store…", StatusColors.Yellow);
        try
        {
            var apps = await StoreUpdatesInfo.ScanAsync();
            IcStore.ItemsSource = apps;
            if (apps == null) SetStatus(TxtNoStore, "Store scan unavailable on this machine (no Store or service disabled).", StatusColors.Muted);
            else if (apps.Count == 0) SetStatus(TxtNoStore, "No pending Store updates.", StatusColors.Muted);
            else TxtNoStore.Visibility = Visibility.Collapsed;
        }
        catch (Exception ex) { SetStatus(TxtNoStore, $"Scan failed: {ex.Message}", StatusColors.Red); }
    }

    private async Task ScanVendorAsync()
    {
        SetStatus(TxtNoVendor, "● Scanning vendor driver/BIOS updates…", StatusColors.Yellow);
        // Auto-scan never installs the OEM tool — it only scans if the tool is already present.
        void OnStatus(string s) => Dispatcher.Invoke(() => SetStatus(TxtNoVendor, "● " + s, StatusColors.Yellow));
        try
        {
            var result = await VendorUpdatesInfo.ScanAsync(OnStatus, installIfMissing: false);
            IcVendor.ItemsSource = result.Updates;
            if (result.Updates.Count == 0) SetStatus(TxtNoVendor, result.Status, StatusColors.Muted);
            else TxtNoVendor.Visibility = Visibility.Collapsed;
        }
        catch (Exception ex) { SetStatus(TxtNoVendor, $"Scan failed: {ex.Message}", StatusColors.Red); }
    }


    // ── UPDATE ALL ────────────────────────────────────────────────────────

    private UpdateTask? _tDefender, _tWinUpdate, _tManufacturer, _tWinget, _tStore, _tOffice;

    private async void UpdateAll_Click(object sender, RoutedEventArgs e)
    {
        // App-wide servicing gate: Update All must not overlap Full Repair / WU Reset / CHKDSK etc.
        // (both sides drive CBS/TrustedInstaller; running them together makes one fail — observed
        // as DISM RestoreHealth 0x800F0915).
        if (!ServicingLock.TryAcquire("Update All"))
        {
            MessageWindow.Show("Please Wait", "Another heavy operation is running",
                $"“{ServicingLock.CurrentOperation}” is still running. Running two Windows " +
                "servicing operations at once makes them fail — wait for it to finish first.",
                MessageKind.Warning, Window.GetWindow(this));
            return;
        }
        // Deliberately NOT tech-gated: keeping machines updated is encouraged, not destructive —
        // users can run the same updates themselves through Windows. Still fully activity-logged.
        ActivityLog.Action("Updates", "Update all (Defender signatures + Windows Update + manufacturer tool + winget upgrade)");
        BtnUpdateAll.IsEnabled = false;
        BtnUpdateAll.Content   = "Running…";
        TxtLog.Text            = "";

        _tDefender    = new UpdateTask { Name = "Windows Defender Signatures" };
        _tWinUpdate   = new UpdateTask { Name = "Windows Update" };
        _tManufacturer = new UpdateTask { Name = _mfrTool?.DisplayName ?? "Manufacturer Update Tool" };
        _tWinget      = new UpdateTask { Name = "App Updates (winget)" };
        _tStore       = new UpdateTask { Name = "Microsoft Store Apps" };
        _tOffice      = new UpdateTask { Name = "Microsoft Office / 365" };

        TaskList.ItemsSource = new[] { _tDefender, _tWinUpdate, _tManufacturer, _tWinget, _tStore, _tOffice };
        PnlTasks.Visibility  = Visibility.Visible;

        try
        {
            await RunDefenderUpdate(_tDefender);
            await RunWindowsUpdate(_tWinUpdate);
            await RunManufacturerUpdate(_tManufacturer);
            await RunWinget(_tWinget);
            await RunStoreUpdate(_tStore);
            await RunOfficeUpdate(_tOffice);
            Log("━━━ All tasks complete ━━━");
        }
        catch (Exception ex)
        {
            Log($"━━━ Stopped — unexpected error: {ex.Message} ━━━");
        }
        finally
        {
            BtnUpdateAll.Content   = "Run Again";
            BtnUpdateAll.IsEnabled = true;
            SaveLog();
            ServicingLock.Release();
        }
    }

    // ── helpers ───────────────────────────────────────────────────────────

    private void Set(UpdateTask task, string text, System.Windows.Media.SolidColorBrush color)
        => Dispatcher.Invoke(() => { task.StatusText = text; task.StatusColor = color; });

    private void SaveLog()
    {
        try
        {
            const string logDir = @"C:\PCI\Logs";
            Directory.CreateDirectory(logDir);

            var timestamp = DateTime.Now.ToString("MMddyyyy_HHmmss");   // seconds — avoid same-minute overwrite
            var path      = Path.Combine(logDir, $"PartnerTool_{Environment.MachineName}_{timestamp}.log");

            var sb = new System.Text.StringBuilder();
            sb.AppendLine("PARTNER TOOL — UPDATE ALL LOG");
            sb.AppendLine($"Device    : {Environment.MachineName}");
            sb.AppendLine($"User      : {Environment.UserName}");
            sb.AppendLine($"Date/Time : {DateTime.Now:MM-dd-yyyy HH:mm:ss}");
            sb.AppendLine(new string('─', 50));
            sb.AppendLine();
            sb.AppendLine("TASK RESULTS");

            foreach (var task in new[] { _tDefender, _tWinUpdate, _tManufacturer, _tWinget, _tStore, _tOffice })
            {
                if (task != null)
                    sb.AppendLine($"  {task.Name,-35} {task.StatusText}");
            }

            sb.AppendLine();
            sb.AppendLine("OUTPUT LOG");
            sb.AppendLine(Dispatcher.Invoke(() => TxtLog.Text));

            File.WriteAllText(path, LogText.ToAscii(sb.ToString()), LogText.Utf8NoBom);   // plain ASCII → no mojibake in tickets
            Log($"Log saved → {path}");
        }
        catch (Exception ex)
        {
            Log($"Could not save log: {ex.Message}");
        }
    }

    private static readonly Regex _ansi = new(@"\x1B\[[0-9;]*[A-Za-z]|\x1B.", RegexOptions.Compiled);

    // Pure download-progress lines, e.g. "1024 KB / 3.00 MB" or a bare "7%"
    private static readonly Regex _progress = new(
        @"^[\d.,]+\s*(?:B|KB|MB|GB|TB)\s*/\s*[\d.,]+\s*(?:B|KB|MB|GB|TB)$|^\d{1,3}\s*%$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);

    private static string? CleanOutput(string? raw)
    {
        if (raw == null) return null;
        // Winget uses \r to overwrite progress lines — keep the last segment
        var line = raw.Contains('\r') ? raw.Split('\r').Last() : raw;
        line = _ansi.Replace(line, "").Replace("\0", "").Trim();
        if (string.IsNullOrWhiteSpace(line)) return null;
        // Filter spinner frames (-  \  |  /) and short noise
        if (line.Length <= 4 && line.All(c => @"-|/\ ".Contains(c))) return null;
        // Filter lines that are mostly progress-bar block characters
        var blocks = line.Count(c => "█░▒▓■□".Contains(c));
        if (blocks > line.Length / 3) return null;
        // Filter leftover download-progress lines
        if (_progress.IsMatch(line)) return null;
        return line;
    }

    private void Log(string line)
        => Dispatcher.Invoke(() =>
        {
            TxtLog.Text += line + "\n";
            if (ChkAutoScroll.IsChecked == true) LogScroll.ScrollToBottom();
        });

    private async Task<int> RunProc(string exe, string args, string tag,
        System.Text.Encoding? outputEncoding = null)
    {
        Log($"▶ {tag}");
        try
        {
            var psi = new ProcessStartInfo(ProcessRunner.ResolveSystemExe(exe), args)
            {
                UseShellExecute        = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow         = true,
            };
            if (outputEncoding != null)
            {
                psi.StandardOutputEncoding = outputEncoding;
                psi.StandardErrorEncoding  = outputEncoding;
            }
            using var p = Process.Start(psi)!;
            p.OutputDataReceived += (_, ev) => { var c = CleanOutput(ev.Data); if (c != null) Log($"  {c}"); };
            p.ErrorDataReceived  += (_, ev) => { var c = CleanOutput(ev.Data); if (c != null) Log($"  {c}"); };
            p.BeginOutputReadLine();
            p.BeginErrorReadLine();
            await p.WaitForExitAsync();
            return p.ExitCode;
        }
        catch (Exception ex)
        {
            Log($"  Error: {ex.Message}");
            return -1;
        }
    }

    // ── individual update tasks ───────────────────────────────────────────

    private async Task RunDefenderUpdate(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);
        var path = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles),
            "Windows Defender", "MpCmdRun.exe");
        if (!File.Exists(path))
        {
            Set(task, "● Not found", StatusColors.Muted);
            Log("Defender: MpCmdRun.exe not found.");
            return;
        }
        var code = await RunProc(path, "-SignatureUpdate", "Defender: updating signatures");
        Set(task, code == 0 ? "● Updated" : "● Failed",
            code == 0 ? StatusColors.Green : StatusColors.Red);
    }

    private async Task RunWindowsUpdate(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);
        Log("Windows Update: using PSWindowsUpdate module...");

        var ps = @"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue | Where-Object { $_.Version -ge '2.8.5.201' })) {
    Write-Output 'Installing NuGet provider...'
    Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope AllUsers | Out-Null
}
if (-not (Get-Module -Name PSWindowsUpdate -ListAvailable)) {
    Write-Output 'Installing PSWindowsUpdate module...'
    Install-Module -Name PSWindowsUpdate -Force -Scope AllUsers -AllowClobber
}
Import-Module PSWindowsUpdate -Force
Write-Output 'Scanning for available updates...'
$updates = Get-WindowsUpdate
if (-not $updates -or $updates.Count -eq 0) {
    Write-Output 'No updates available. System is up to date.'
    exit 0
}
Write-Output ""Found $($updates.Count) update(s):""
$updates | ForEach-Object { Write-Output ""  [$($_.KB)] $($_.Title)"" }
Write-Output 'Installing...'
# Format the progress rows ourselves rather than letting PSWindowsUpdate print its default table.
# Its Size column is WUA's MaxDownloadSize - the maximum POSSIBLE payload, which for a UUP cumulative
# update is the whole range (the infamous ""91GB"" for a ~400 MB delta). There is no trustworthy real
# size to print in its place, so we print none: result, KB and title are what the tech acts on.
# Objects stream as each update is Accepted / Downloaded / Installed, so this stays live.
Install-WindowsUpdate -AcceptAll -IgnoreReboot -AutoReboot:$false *>&1 | ForEach-Object {
    if ($_.PSObject.Properties['Result'] -and $_.PSObject.Properties['KB']) {
        Write-Output (""  {0,-10} [{1}] {2}"" -f $_.Result, $_.KB, $_.Title)
    } else {
        Write-Output $_          # warnings / verbose text (e.g. 'Reboot is required') pass through
    }
}
Write-Output 'Done.'
";
        var script = Path.Combine(Path.GetTempPath(), "pci_wu_update.ps1");
        try
        {
            await File.WriteAllTextAsync(script, ps);
            var code = await RunProc("powershell.exe",
                $"-ExecutionPolicy Bypass -NonInteractive -File \"{script}\"",
                "PSWindowsUpdate: scanning and installing");
            Set(task, code == 0 ? "● Updated" : "● Done (check log)",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        finally { try { File.Delete(script); } catch { } }
    }

    private async Task RunManufacturerUpdate(UpdateTask task)
    {
        if (_mfrTool == null)
        {
            Set(task, "● Unsupported", StatusColors.Muted);
            Log("Manufacturer: no known update tool detected.");
            return;
        }

        var displayName = _mfrTool.DisplayName.ToLowerInvariant();

        if (displayName.Contains("dell"))
        {
            await RunDellUpdate(task);
        }
        else if (displayName.Contains("lenovo"))
        {
            await RunLenovoUpdate(task);
        }
        else if (displayName.Contains("hp") || displayName.Contains("hewlett"))
        {
            await RunHpUpdate(task);
        }
        else
        {
            Log($"{_mfrTool.DisplayName}: no silent CLI available — opening app.");
            Dispatcher.Invoke(() =>
            {
                try
                {
                    var target = _mfrExe ?? _mfrTool.StoreUri;
                    if (target != null)
                        Process.Start(new ProcessStartInfo(target) { UseShellExecute = true });
                }
                catch { }
            });
            Set(task, "● Opened", StatusColors.Blue);
        }
    }

    private static string? FindDellCli() => new[]
    {
        @"C:\Program Files\Dell\CommandUpdate\dcu-cli.exe",
        @"C:\Program Files (x86)\Dell\CommandUpdate\dcu-cli.exe",
    }.FirstOrDefault(File.Exists);

    private async Task RunDellUpdate(UpdateTask task)
    {
        // Prefer the CLI executable — DCUx86.exe is the UI, dcu-cli.exe is the headless runner
        var cliExe = _mfrExe?.EndsWith("dcu-cli.exe", StringComparison.OrdinalIgnoreCase) == true
            ? _mfrExe
            : FindDellCli();

        if (cliExe == null)
        {
            // Mirror the HUS script: auto-install Dell Command Update via winget, then re-resolve.
            // (Update All is tech-gated, so installing the prerequisite here is expected behavior —
            // only the passive scan-on-open never installs anything.)
            Set(task, "Installing DCU…", StatusColors.Yellow);
            Log("Dell: dcu-cli.exe not found — installing Dell Command Update via winget (one-time, can take a few minutes)…");
            ActivityLog.Action("Updates", "Auto-install Dell Command Update (winget Dell.CommandUpdate.Universal)");
            try { await VendorUpdatesInfo.InstallViaWingetAsync("Dell.CommandUpdate.Universal"); } catch { }
            cliExe = FindDellCli();
            if (cliExe != null) Log($"Dell: installed — CLI at {cliExe}.");
        }

        if (cliExe == null)
        {
            // Couldn't install (winget unavailable / blocked) — fall back to the GUI as a last resort.
            Log("Dell: couldn't install Dell Command Update automatically — opening the GUI instead.");
            Dispatcher.Invoke(() =>
            {
                if (_mfrExe != null)
                    try { Process.Start(new ProcessStartInfo(_mfrExe) { UseShellExecute = true }); } catch { }
            });
            Set(task, "● Opened (install failed)", StatusColors.Yellow);
            return;
        }

        Set(task, "Running…", StatusColors.Yellow);
        var code = await RunProc(cliExe,
            "/applyUpdates -updateType=bios,driver,firmware,application,utility -reboot=disable",
            "Dell Command Update: applying updates");

        Set(task, code switch
        {
            0   => "● Updated",
            1   => "● Updated — Reboot Required",
            5   => "● Reboot required from a prior update — rerun after restart",
            500 => "● Up to date",
            501 => "● Up to date",
            _   => "● Done (check log)"
        },
        code switch
        {
            0 or 500 or 501 => StatusColors.Green,
            1 or 5          => StatusColors.Yellow,
            _               => StatusColors.Yellow
        });
    }

    private async Task RunLenovoUpdate(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);
        Log("Lenovo: using LSUClient PowerShell module (only unattended packages)...");

        // LSUClient is a PS module designed for headless RMM contexts — much more reliable than tvsu.exe
        var ps = @"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue | Where-Object { $_.Version -ge '2.8.5.201' })) {
    Write-Output 'Installing NuGet provider...'
    Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope CurrentUser | Out-Null
}
if (-not (Get-Module -Name LSUClient -ListAvailable)) {
    Write-Output 'Installing LSUClient module...'
    Install-Module -Name LSUClient -Force -Scope CurrentUser -AllowClobber
}
Import-Module LSUClient -Force
Write-Output 'Scanning for updates...'
$updates = Get-LSUpdate | Where-Object { $_.Installer.Unattended }
if (-not $updates -or $updates.Count -eq 0) { Write-Output 'No applicable updates found.'; exit 0 }
Write-Output ""Found $($updates.Count) update(s):""
$updates | ForEach-Object { Write-Output ""  - $($_.Title)"" }
Write-Output 'Downloading...'
$updates | Save-LSUpdate -Path ""$env:TEMP\LsuUpdates""
Write-Output 'Installing...'
$results = $updates | Install-LSUpdate
foreach ($r in $results) { Write-Output ""[$($r.Result)] $($r.Title)"" }
";
        var scriptPath = Path.Combine(Path.GetTempPath(), "pci_lsu_update.ps1");
        try
        {
            await File.WriteAllTextAsync(scriptPath, ps);
            var code = await RunProc("powershell.exe",
                $"-ExecutionPolicy Bypass -NonInteractive -File \"{scriptPath}\"",
                "Lenovo LSUClient: running updates");
            Set(task, code == 0 ? "● Updated" : "● Done (check log)",
                code == 0 ? StatusColors.Green : StatusColors.Yellow);
        }
        finally
        {
            try { File.Delete(scriptPath); } catch { }
        }
    }

    private async Task RunHpUpdate(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);

        const string hpiaPath = @"C:\PCI\Tools\HPImageAssistant.exe";
        if (!File.Exists(hpiaPath))
        {
            // Mirror the HUS script: fetch HP Image Assistant from HP when it's missing.
            // Harden the folder first — we're downloading an exe we'll run elevated.
            Set(task, "Downloading HPIA…", StatusColors.Yellow);
            Log("HP: HP Image Assistant not found — downloading from HP (one-time, can take a few minutes)…");
            ActivityLog.Action("Updates", "Auto-download HP Image Assistant to C:\\PCI\\Tools");
            SecureDirectory.EnsureHardened(@"C:\PCI\Tools");
            try { await VendorUpdatesInfo.EnsureHpiaAsync(); } catch { }
            if (File.Exists(hpiaPath)) Log("HP: HP Image Assistant ready.");
        }

        if (File.Exists(hpiaPath))
        {
            Log("HP: running HP Image Assistant...");
            var reportDir = Path.Combine(Path.GetTempPath(), "HPIA_Report");
            Directory.CreateDirectory(reportDir);

            var code = await RunProc(hpiaPath,
                $"/Operation:Analyze /Action:Install /Silent /Category:All /ReportFolder:\"{reportDir}\" /SoftPaqFolder:\"{reportDir}\" /NoReboot",
                "HP Image Assistant: installing updates");

            Set(task, code switch { 0 => "● Updated", 1 => "● Up to date", 2 => "● Reboot needed", _ => "● Done (check log)" },
                code is 0 or 1 or 2 ? StatusColors.Green : StatusColors.Yellow);
        }
        else if (_mfrExe != null)
        {
            // Download failed (connectivity / HP page changed) — open HP Support Assistant as last resort.
            Log("HP: couldn't download HP Image Assistant — opening HP Support Assistant instead.");
            Dispatcher.Invoke(() =>
            {
                try { Process.Start(new ProcessStartInfo(_mfrExe) { UseShellExecute = true }); } catch { }
            });
            Set(task, "● Opened (download failed)", StatusColors.Yellow);
        }
        else
        {
            Set(task, "● Unavailable", StatusColors.Muted);
            Log("HP: couldn't download HP Image Assistant and HP Support Assistant isn't installed.");
        }
    }

    private async Task RunWinget(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);
        var winget = WingetLocator.Path();
        if (winget.Length == 0)
        {
            Log("▶ winget: skipped");
            Log("  " + WingetLocator.Unavailable);
            Set(task, "● Skipped — winget unavailable", StatusColors.Yellow);
            return;
        }

        var code = await RunProc(winget,
            "upgrade --all --accept-source-agreements --accept-package-agreements --disable-interactivity",
            "winget: upgrading all packages",
            System.Text.Encoding.UTF8);
        Set(task, code == 0 ? "● Done" : "● Done (some errors)",
            code == 0 ? StatusColors.Green : StatusColors.Yellow);
    }

    private async Task RunStoreUpdate(UpdateTask task)
    {
        Set(task, "Running…", StatusColors.Yellow);
        Log("Store: checking for updates via WinRT...");
        try
        {
            var manager = new Windows.ApplicationModel.Store.Preview.InstallControl.AppInstallManager();
            var updates  = await manager.SearchForAllUpdatesAsync();

            // A fresh search misses updates the Store already queued itself (stuck at
            // ReadyToDownload, or errored with "app was in use") — pick those up from the
            // install queue and restart them so they retry now.
            var pendingNames = updates.Select(u => u.PackageFamilyName)
                                      .ToHashSet(StringComparer.OrdinalIgnoreCase);
            int restarted = 0;
            foreach (var item in manager.AppInstallItemsWithGroupSupport)
            {
                if (item.InstallType != Windows.ApplicationModel.Store.Preview.InstallControl.AppInstallType.Update)
                    continue;
                AppInstallState state;
                try { state = item.GetCurrentStatus().InstallState; } catch { continue; }
                if (state is AppInstallState.Completed or AppInstallState.Canceled) continue;
                pendingNames.Add(item.PackageFamilyName);
                if (state is AppInstallState.Error or AppInstallState.Paused
                          or AppInstallState.PausedLowBattery or AppInstallState.PausedWiFiRecommended
                          or AppInstallState.PausedWiFiRequired or AppInstallState.ReadyToDownload)
                {
                    try { item.Restart(); restarted++; } catch { }
                }
            }

            int total = pendingNames.Count;
            if (total == 0)
            {
                Set(task, "● Up to date", StatusColors.Green);
                Log("Store: no pending updates.");
                return;
            }

            Log($"Store: {total} update(s) queued" +
                (restarted > 0 ? $" ({restarted} restarted from a stuck state):" : ":"));
            foreach (var name in pendingNames.OrderBy(n => n, StringComparer.OrdinalIgnoreCase))
                Log($"  {name.Split('_')[0]}");

            Log("Monitoring download progress...");
            var hardTimeout  = DateTime.Now.AddMinutes(10);
            var stallTimeout = TimeSpan.FromSeconds(90);
            var lastChange   = DateTime.Now;
            string lastLine  = "";

            while (DateTime.Now < hardTimeout)
            {
                await Task.Delay(4000);
                var active = manager.AppInstallItemsWithGroupSupport;

                // Items drop off the queue once they finish/are removed → everything done.
                if (active.Count == 0)
                {
                    Set(task, $"● {total} updated", StatusColors.Green);
                    Log("Store: all updates completed.");
                    return;
                }

                var statuses = active.Select(i =>
                {
                    var op = i.GetCurrentStatus();
                    return (Name: i.PackageFamilyName.Split('_')[0],
                            State: op.InstallState,
                            Pct: op.PercentComplete,
                            Err: op.ErrorCode?.HResult ?? 0);
                }).ToList();

                // Every remaining item has reached a final state — stop now rather than
                // waiting out the full timeout (errored items linger in the queue).
                bool allFinal = statuses.All(s =>
                    s.State is AppInstallState.Completed
                            or AppInstallState.Error
                            or AppInstallState.Canceled);
                if (allFinal)
                {
                    var failed = statuses.Where(s => s.State != AppInstallState.Completed).ToList();
                    if (failed.Count == 0)
                    {
                        Set(task, $"● {total} updated", StatusColors.Green);
                        Log("Store: all updates completed.");
                    }
                    else
                    {
                        Set(task, $"● {total - failed.Count}/{total} updated, {failed.Count} failed",
                            StatusColors.Yellow);
                        const int packagesInUse = unchecked((int)0x80073D02);   // ERROR_PACKAGES_IN_USE
                        foreach (var s in failed)
                            Log(s.State == AppInstallState.Error && s.Err == packagesInUse
                                ? $"  {s.Name}: app was in use — the Store retries once it closes (or after a reboot)"
                                : $"  {s.Name}: {s.State}" + (s.Err != 0 ? $" (0x{s.Err:X8})" : ""));
                    }
                    return;
                }

                var line = string.Join(" | ", statuses.Select(s => $"{s.Name}: {s.State} {s.Pct}%"));
                if (line != lastLine)
                {
                    Log($"  {line}");
                    lastLine   = line;
                    lastChange = DateTime.Now;
                }
                else if (DateTime.Now - lastChange > stallTimeout)
                {
                    // No item advanced for 90s — something is wedged (e.g. stuck at
                    // ReadyToDownload). Don't block the rest of Update All.
                    Set(task, "● Stalled (check Store)", StatusColors.Yellow);
                    Log($"Store: no progress for {stallTimeout.TotalSeconds:F0}s — skipping. Last state: {line}");
                    return;
                }
            }

            Set(task, "● Timed out (check Store)", StatusColors.Yellow);
            Log("Store: timed out — check Microsoft Store for remaining updates.");
        }
        catch (Exception ex)
        {
            Log($"Store: {ex.GetType().Name} — opening Store updates page.");
            Dispatcher.Invoke(() =>
                Process.Start(new ProcessStartInfo("ms-windows-store://downloadsandupdates")
                    { UseShellExecute = true }));
            Set(task, "● Opened Store", StatusColors.Blue);
        }
    }

    private async Task RunOfficeUpdate(UpdateTask task)
    {
        var c2r = new[]
        {
            @"C:\Program Files\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe",
            @"C:\Program Files (x86)\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe",
        }.FirstOrDefault(File.Exists);

        if (c2r == null)
        {
            Set(task, "● Not installed", StatusColors.Muted);
            Log("Office: not found, skipping.");
            return;
        }

        // Hand off to Office's own Click-to-Run updater and let it show its normal progress window
        // — the previous silent registry-polling approach was unreliable (wrong "done"/timeout
        // status). It pops up, downloads and installs on its own; we just launch it and move on.
        Set(task, "Launching…", StatusColors.Yellow);
        // The C2R updater checks/downloads silently first, so its window often doesn't appear until
        // MINUTES after everything else here reads done — say so, or the popup gets reported as a
        // mystery leak by whoever has moved on to something else (field report, 0.19.10).
        Log("Office: launching the built-in Click-to-Run updater — its progress window may not appear for a few minutes.");
        try
        {
            // displaylevel=true forces the visible update UI even when run elevated;
            // forceappshutdown=false lets it wait for open Office apps instead of closing them.
            Process.Start(new ProcessStartInfo(c2r, "/update user displaylevel=true forceappshutdown=false")
            {
                UseShellExecute = false,
            });
            ActivityLog.Command("Updates", "OfficeC2RClient.exe", "/update user displaylevel=true");
            Log("Office: updater launched.");
            Set(task, "● Launched — Office's own window may appear a few minutes later", StatusColors.Blue);
        }
        catch (Exception ex)
        {
            Log($"Office: couldn't launch the updater — {ex.Message}");
            Set(task, "● Failed to launch", StatusColors.Red);
        }
        await Task.CompletedTask;
    }
}

public class UpdateTask : System.ComponentModel.INotifyPropertyChanged
{
    private string                       _statusText  = "Pending";
    private System.Windows.Media.Brush   _statusColor = StatusColors.Muted;

    public required string Name { get; init; }

    public string StatusText
    {
        get => _statusText;
        set { _statusText = value; OnPropertyChanged(); }
    }

    public System.Windows.Media.Brush StatusColor
    {
        get => _statusColor;
        set { _statusColor = value; OnPropertyChanged(); }
    }

    public event System.ComponentModel.PropertyChangedEventHandler? PropertyChanged;
    void OnPropertyChanged([System.Runtime.CompilerServices.CallerMemberName] string? p = null)
        => PropertyChanged?.Invoke(this, new(p));
}
```
