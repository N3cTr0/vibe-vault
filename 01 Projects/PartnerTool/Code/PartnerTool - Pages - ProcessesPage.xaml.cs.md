---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ProcessesPage.xaml.cs
---

# PartnerTool\Pages\ProcessesPage.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Threading;

namespace PartnerTool.Pages;

public partial class ProcessesPage : UserControl
{
    private readonly ProcessSampler _sampler = new();
    private readonly DispatcherTimer _timer = new() { Interval = TimeSpan.FromSeconds(2) };
    private List<ProcRow> _rows = new();
    private bool _sampling;
    private bool _paused;
    private string _sortKey = "cpu";
    private bool _sortAsc;   // numeric columns default to descending (like clicking CPU in taskmgr)

    public ProcessesPage()
    {
        InitializeComponent();
        UpdateSortHeaders();
        _timer.Tick += async (_, _) => await SampleAsync();
        // Only burn cycles while the page is actually on screen.
        IsVisibleChanged += async (_, _) =>
        {
            if (IsVisible) { if (!_paused) _timer.Start(); await SampleAsync(); }
            else _timer.Stop();
        };
    }

    private async Task SampleAsync()
    {
        if (_sampling) return;   // a slow sample must not stack behind the next tick
        _sampling = true;
        try
        {
            _rows = await Task.Run(_sampler.Sample);
            ApplyView();
        }
        catch { /* transient — try again next tick */ }
        finally { _sampling = false; }
    }

    private void Pause_Click(object sender, RoutedEventArgs e)
    {
        _paused = !_paused;
        BtnPause.Content = _paused ? "Resume" : "Pause";
        if (_paused) _timer.Stop();
        else { _timer.Start(); _ = SampleAsync(); }
    }

    private void ProcSearch_TextChanged(object sender, TextChangedEventArgs e) => ApplyView();

    // ── Sorting (click the column headers, like Task Manager) ─────────────

    private static readonly string[] NumericKeys = ["cpu", "mem", "disk", "threads", "handles", "pid"];

    private void Sort_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: string key }) return;
        if (_sortKey == key) _sortAsc = !_sortAsc;
        else { _sortKey = key; _sortAsc = !NumericKeys.Contains(key); }   // numbers start descending
        UpdateSortHeaders();
        ApplyView();
    }

    private void UpdateSortHeaders()
    {
        void Set(Button b, string key, string label)
            => b.Content = _sortKey == key ? $"{label} {(_sortAsc ? "▲" : "▼")}" : label;
        Set(ColName,    "name",    "Name");
        Set(ColPid,     "pid",     "PID");
        Set(ColStatus,  "status",  "Status");
        Set(ColUser,    "user",    "User");
        Set(ColCpu,     "cpu",     "CPU");
        Set(ColMem,     "mem",     "Memory");
        Set(ColDisk,    "disk",    "Disk");
        Set(ColThreads, "threads", "Threads");
        Set(ColHandles, "handles", "Handles");
    }

    private void ApplyView()
    {
        IEnumerable<ProcRow> items = _sortKey switch
        {
            "pid"     => _rows.OrderBy(r => r.Pid),
            "status"  => _rows.OrderBy(r => r.Status).ThenBy(r => r.Name, StringComparer.OrdinalIgnoreCase),
            "user"    => _rows.OrderBy(r => r.User, StringComparer.OrdinalIgnoreCase).ThenBy(r => r.Name, StringComparer.OrdinalIgnoreCase),
            "cpu"     => _rows.OrderBy(r => r.Cpu),
            "mem"     => _rows.OrderBy(r => r.MemMb),
            "disk"    => _rows.OrderBy(r => r.DiskMbs),
            "threads" => _rows.OrderBy(r => r.Threads),
            "handles" => _rows.OrderBy(r => r.Handles),
            _         => _rows.OrderBy(r => r.Name, StringComparer.OrdinalIgnoreCase),
        };
        if (!_sortAsc) items = items.Reverse();

        var q = TxtProcSearch.Text?.Trim() ?? "";
        if (!string.IsNullOrEmpty(q))
            items = items.Where(r => r.Name.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     r.User.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     r.Desc.Contains(q, StringComparison.OrdinalIgnoreCase) ||
                                     r.Pid.ToString().Contains(q));

        var shown = items.ToList();

        // Preserve scroll position + selection across the 2 s refresh (keyed by PID).
        int? selPid = (LstProcs.SelectedItem as ProcRow)?.Pid;
        LstProcs.ItemsSource = shown;
        if (selPid is { } pid && shown.FirstOrDefault(r => r.Pid == pid) is { } again)
            LstProcs.SelectedItem = again;

        TxtProcCount.Text = string.IsNullOrEmpty(q)
            ? $"{_rows.Count} processes"
            : $"{shown.Count} of {_rows.Count}";
    }

    // ── End Task ──────────────────────────────────────────────────────────

    private async void EndTask_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: ProcRow row }) return;
        if (!row.CanEnd) return;   // belt & braces — the button is disabled for protected procs

        if (!MessageWindow.Confirm("End Task", $"End “{row.Name}” (PID {row.Pid})?",
                "Unsaved work in this program will be lost. Ending a process the system depends on can " +
                "make Windows unstable until a restart.", MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Processes", $"End task {row.Name} (PID {row.Pid})");
        bool ok = await Task.Run(() => ProcessInfo.TryKill(row.Pid));
        ActivityLog.Result("Processes", $"{row.Name} (PID {row.Pid}): {(ok ? "ended" : "could not be ended")}");
        if (!ok)
            MessageWindow.Show("End Task", $"Couldn't end {row.Name}",
                "The process may have already exited, or Windows refused (protected/system process).",
                MessageKind.Warning, Window.GetWindow(this));
        await SampleAsync();
    }
}
```
