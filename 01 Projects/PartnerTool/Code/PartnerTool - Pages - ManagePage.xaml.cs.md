---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ManagePage.xaml.cs
---

# PartnerTool\Pages\ManagePage.xaml.cs

```csharp
using System.Diagnostics;
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class ManagePage : UserControl
{
    private List<ServiceItem> _allServices = new();
    private readonly HashSet<string> _loaded = new();
    private string _section = "Services";

    public ManagePage()
    {
        InitializeComponent();
        UpdateSortHeaders();
        IsVisibleChanged += (_, _) => { if (IsVisible) ShowSection(_section); };
    }

    /// <summary>
    /// Warm the two slowest sections (Services + Scheduled Tasks) in the background at startup so
    /// the first jump into Manage — and switching between its sections — is instant. Marks them
    /// loaded so a later ShowSection won't collect them again.
    /// </summary>
    private static readonly string[] AllSections =
        { "Services", "Tasks", "Drivers", "Hosts", "Env", "Features", "Profiles" };

    public async Task PreloadAsync()
    {
        // Warm EVERY section in the background so any jump into Manage — and switching between its
        // sections — is instant, not just Services/Tasks.
        foreach (var tag in AllSections)
        {
            try { await EnsureLoaded(tag); }
            catch { /* best-effort; the on-demand path still works */ }
        }
    }

    private void Sub_Click(object sender, RoutedEventArgs e)
    {
        if (sender is Button { Tag: string tag }) ShowSection(tag);
    }

    private async void ShowSection(string tag)
    {
        _section = tag;
        SecServices.Visibility = tag == "Services" ? Visibility.Visible : Visibility.Collapsed;
        SecTasks.Visibility    = tag == "Tasks"    ? Visibility.Visible : Visibility.Collapsed;
        SecDrivers.Visibility  = tag == "Drivers"  ? Visibility.Visible : Visibility.Collapsed;
        SecHosts.Visibility    = tag == "Hosts"    ? Visibility.Visible : Visibility.Collapsed;
        SecEnv.Visibility      = tag == "Env"      ? Visibility.Visible : Visibility.Collapsed;
        SecFeatures.Visibility = tag == "Features" ? Visibility.Visible : Visibility.Collapsed;
        SecProfiles.Visibility = tag == "Profiles" ? Visibility.Visible : Visibility.Collapsed;

        await EnsureLoaded(tag);
    }

    /// <summary>Collect a section's data once (used by both on-demand navigation and the startup preload).</summary>
    private async Task EnsureLoaded(string tag)
    {
        if (!_loaded.Add(tag)) return;   // HashSet.Add is false if already loaded
        switch (tag)
        {
            case "Services": await LoadServices(); break;
            case "Tasks":    await LoadTasks(); break;
            case "Drivers":  await LoadDrivers(); break;
            case "Hosts":    HostsReload_Click(this, new RoutedEventArgs()); break;
            case "Env":      LstEnv.ItemsSource = await Task.Run(SystemManagement.EnvVars); break;
            case "Features": LstFeatures.ItemsSource = await Task.Run(SystemManagement.OptionalFeatures); break;
            case "Profiles": await LoadProfiles(); break;
        }
    }

    // ── Services ─────────────────────────────────────────────
    private async Task LoadServices()
    {
        TxtSvcStatus.Text = "Loading…";
        _allServices = await Task.Run(ServicesInfo.Collect);
        FilterServices();
        TxtSvcStatus.Text = $"{_allServices.Count} services";
    }

    private void SvcSearch_TextChanged(object sender, TextChangedEventArgs e) => FilterServices();

    // ── Sorting (click the headers, like services.msc) ──────
    private string _sortKey = "name";
    private bool   _sortAsc = true;

    private void SortServices_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: string key }) return;
        if (_sortKey == key) _sortAsc = !_sortAsc;          // same header → flip direction
        else { _sortKey = key; _sortAsc = true; }
        UpdateSortHeaders();
        FilterServices();
    }

    private void UpdateSortHeaders()
    {
        void Set(Button b, string key, string label)
            => b.Content = _sortKey == key ? $"{label} {(_sortAsc ? "▲" : "▼")}" : label;
        Set(BtnSortName,    "name",    "Name");
        Set(BtnSortStatus,  "status",  "Status");
        Set(BtnSortStartup, "startup", "Startup");
    }

    private static int StartOrder(string mode) => mode switch
    {
        "Auto" or "Automatic" => 0,
        "Manual"              => 1,
        "Disabled"            => 2,
        _                     => 3,
    };

    private void FilterServices()
    {
        IEnumerable<ServiceItem> items = _sortKey switch
        {
            "status"  => _allServices.OrderBy(s => s.Running ? 0 : 1).ThenBy(s => s.DisplayName, StringComparer.OrdinalIgnoreCase),
            "startup" => _allServices.OrderBy(s => StartOrder(s.StartMode)).ThenBy(s => s.DisplayName, StringComparer.OrdinalIgnoreCase),
            _         => _allServices.OrderBy(s => s.DisplayName, StringComparer.OrdinalIgnoreCase),
        };
        if (!_sortAsc) items = items.Reverse();

        var q = TxtSvcSearch.Text?.Trim() ?? "";
        if (!string.IsNullOrEmpty(q))
            items = items.Where(s => s.DisplayName.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     s.Name.Contains(q, StringComparison.OrdinalIgnoreCase));

        var shown = items.ToList();
        LstServices.ItemsSource = shown;
        // Reflect the filter in the count so a search doesn't still read "276 services".
        TxtSvcStatus.Text = string.IsNullOrEmpty(q)
            ? $"{_allServices.Count} services"
            : $"{shown.Count} of {_allServices.Count} services";
    }

    private async void SvcStart_Click(object sender, RoutedEventArgs e)   => await Control(sender, "start", false);
    private async void SvcStop_Click(object sender, RoutedEventArgs e)    => await Control(sender, "stop", true);
    private async void SvcRestart_Click(object sender, RoutedEventArgs e) => await Control(sender, "restart", true);

    private async Task Control(object sender, string action, bool confirm)
    {
        if (sender is not Button { Tag: string name }) return;

        if ((action is "stop" or "restart") && ServicesInfo.IsCritical(name))
        {
            MessageWindow.Show("Services", $"“{name}” is a critical system service",
                "Stopping it can make Windows unusable, so it's blocked here. Use it only via services.msc if you really must.",
                MessageKind.Warning, Window.GetWindow(this));
            return;
        }

        if (confirm && !TechGate.Verify(Window.GetWindow(this))) return;
        if (confirm && !MessageWindow.Confirm("Services", $"{char.ToUpper(action[0]) + action[1..]} “{name}”?",
                "Stopping or restarting a service can disrupt Windows or running apps. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        TxtSvcStatus.Text = $"{action}…";
        ActivityLog.Action("Services", $"{action} service {name}");
        var (ok, msg) = await Task.Run(() => ServicesInfo.Control(name, action));
        ActivityLog.Result("Services", $"{name}: {msg}");
        TxtSvcStatus.Text = $"{name}: {msg}";
        await LoadServices();
    }

    // ── Tasks ────────────────────────────────────────────────
    private List<ScheduledTaskItem> _allTasks = new();

    private async Task LoadTasks()
    {
        TxtTasksStatus.Text = "Loading…";
        _allTasks = await ScheduledTasksInfo.CollectAsync();
        FilterTasks();
    }

    private void TaskSearch_TextChanged(object sender, TextChangedEventArgs e) => FilterTasks();

    private void FilterTasks()
    {
        var q = TxtTaskSearch.Text?.Trim() ?? "";
        var shown = string.IsNullOrEmpty(q)
            ? _allTasks
            : _allTasks.Where(t => t.Name.Contains(q, StringComparison.OrdinalIgnoreCase)).ToList();
        LstTasks.ItemsSource = shown;
        TxtTasksStatus.Text = _allTasks.Count == 0
            ? "No scheduled tasks found (or they couldn't be read)"
            : string.IsNullOrEmpty(q)
                ? $"{_allTasks.Count} scheduled tasks"
                : $"{shown.Count} of {_allTasks.Count} scheduled tasks";
    }

    // ── Drivers ──────────────────────────────────────────────
    private List<DriverItem> _allDrivers = new();

    private async Task LoadDrivers()
    {
        TxtDriversStatus.Text = "Loading…";
        _allDrivers = await Task.Run(DriversInfo.Collect);
        FilterDrivers();
    }

    private void DriverSearch_TextChanged(object sender, TextChangedEventArgs e) => FilterDrivers();

    private void FilterDrivers()
    {
        var q = TxtDriverSearch.Text?.Trim() ?? "";
        var shown = string.IsNullOrEmpty(q)
            ? _allDrivers
            : _allDrivers.Where(d => d.Device.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     d.Provider.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     d.Version.Contains(q, StringComparison.OrdinalIgnoreCase)).ToList();
        LstDrivers.ItemsSource = shown;
        int unsigned = _allDrivers.Count(d => d.Signed == false);
        var suffix = unsigned > 0 ? $"  ·  {unsigned} unsigned" : "";
        TxtDriversStatus.Text = string.IsNullOrEmpty(q)
            ? $"{_allDrivers.Count} drivers{suffix}"
            : $"{shown.Count} of {_allDrivers.Count} drivers{suffix}";
    }

    // ── Hosts ────────────────────────────────────────────────
    private void HostsReload_Click(object sender, RoutedEventArgs e)
    {
        TxtHosts.Text = SystemManagement.ReadHosts();
        TxtHostsStatus.Text = SystemManagement.HostsPath;
    }

    private void HostsSave_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!MessageWindow.Confirm("Hosts File", "Save the hosts file?",
                "This overwrites the system hosts file. A bad entry can break name resolution. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;
        ActivityLog.Action("Hosts", $"Save hosts file ({SystemManagement.HostsPath})");
        var (ok, msg) = SystemManagement.SaveHosts(TxtHosts.Text);
        ActivityLog.Result("Hosts", ok ? "saved" : msg);
        TxtHostsStatus.Text = ok ? "● Saved" : $"● {msg}";
    }

    // ── Environment variables ────────────────────────────────
    private async void EnvAdd_Click(object sender, RoutedEventArgs e)
    {
        bool machine = (CmbEnvScope.SelectedItem as ComboBoxItem)?.Content as string == "System";
        string name  = TxtEnvName.Text;
        string value = TxtEnvValue.Text;

        // Machine scope changes the environment for every user — gate it. (User scope too, for
        // consistency with the tool's other system-changing actions.)
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (machine && !MessageWindow.Confirm("Environment Variable",
                $"Add the System variable “{name.Trim()}”?",
                "System (Machine) variables apply to all users. If it already exists it will be overwritten. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        var (ok, msg) = SystemManagement.SetEnvVar(name, value, machine);
        TxtEnvStatus.Text = (ok ? "● " : "● ") + msg;
        TxtEnvStatus.Foreground = ok ? StatusColors.Green : StatusColors.Red;
        if (ok)
        {
            TxtEnvName.Clear();
            TxtEnvValue.Clear();
            LstEnv.ItemsSource = await Task.Run(SystemManagement.EnvVars);   // reflect the new value
        }
    }

    // ── Features ─────────────────────────────────────────────
    private void OpenFeatures_Click(object sender, RoutedEventArgs e)
    {
        try { Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemTool("optionalfeatures.exe")) { UseShellExecute = true }); }
        catch (Exception ex) { MessageWindow.Show("Features", "Couldn't open", ex.Message, MessageKind.Error, Window.GetWindow(this)); }
    }

    // ── Profiles ─────────────────────────────────────────────
    private async Task LoadProfiles()
    {
        var profiles = await Task.Run(SystemManagement.UserProfiles);
        IcProfiles.ItemsSource = profiles;
        TxtNoProfiles.Visibility = profiles.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
    }

    private async void DeleteProfile_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: UserProfileItem p }) return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;

        if (!MessageWindow.Confirm("Delete Profile", $"Permanently delete the profile for “{p.Name}”?",
                $"This removes {p.Path} and everything in it (desktop, documents, settings) AND the user's " +
                "profile registry entry. It cannot be undone — only do this if they no longer use this machine.",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Profiles", $"Delete user profile {p.Name} (SID {p.Sid}) — {p.Path}");
        var (ok, msg) = await Task.Run(() => SystemManagement.DeleteUserProfile(p.Sid));
        ActivityLog.Result("Profiles", ok ? "removed" : msg);
        MessageWindow.Show("Delete Profile", ok ? "Profile removed" : "Couldn't remove the profile",
            msg, ok ? MessageKind.Info : MessageKind.Error, Window.GetWindow(this));
        await LoadProfiles();
    }
}
```
