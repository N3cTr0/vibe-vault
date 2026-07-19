---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\DiskUsagePage.xaml.cs
---

# PartnerTool\Pages\DiskUsagePage.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Threading;
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class DiskUsagePage : UserControl
{
    private string? _currentPath;
    private CancellationTokenSource? _cts;

    // Fast in-memory MFT tree for the selected drive (null → fall back to the folder walk).
    private MftVolume? _mft;
    private string?    _mftRoot;

    private List<UsageEntry> _items = new();   // the current folder's UNFILTERED children (sizes include hidden)
    private string _resultPrefix = "";         // "⚡" (fast) / "●" (walk) prefix for the result status line

    // ── Column sorting ──
    // Base header captions, in GridView column order. The live headers get a ▲/▼ appended.
    private static readonly string[] HeaderText = { "Name", "% of Parent", "Size", "Files", "Modified", "Attr" };
    // Columns that read largest/newest-first the first time you click them.
    private static readonly bool[] DescFirst = { false, true, true, true, true, false };

    private int  _sortCol  = 1;      // "% of Parent" — the WizTree default
    private bool _sortDesc = true;

    public DiskUsagePage()
    {
        InitializeComponent();
        Loaded += (_, _) => { if (PnlDrives.Children.Count == 0) BuildDrives(); };
        // Free the (potentially large) MFT tree when the page is left.
        IsVisibleChanged += (_, _) => { if (!IsVisible) { _mft = null; _mftRoot = null; } };
        UpdateHeaders();
    }

    /// <summary>Click a header to sort by it; click the same one again to reverse.</summary>
    private void Header_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not GridViewColumnHeader { Column: { } col }) return;
        if (LstEntries.View is not GridView gv) return;

        int i = gv.Columns.IndexOf(col);
        if (i < 0) return;

        if (i == _sortCol) _sortDesc = !_sortDesc;
        else { _sortCol = i; _sortDesc = DescFirst[i]; }

        UpdateHeaders();
        ApplyView();
    }

    private void UpdateHeaders()
    {
        if (LstEntries.View is not GridView gv) return;
        for (int i = 0; i < gv.Columns.Count && i < HeaderText.Length; i++)
            gv.Columns[i].Header = i == _sortCol
                ? HeaderText[i] + (_sortDesc ? "  ▼" : "  ▲")
                : HeaderText[i];
    }

    /// <summary>
    /// Sort the children by the active column. "% of Parent" and "Size" are the same ordering
    /// (percent is derived from bytes), and every sort falls back to size so equal keys stay useful.
    /// The synthetic ".." row is never sorted — <see cref="ApplyView"/> pins it to the top.
    /// </summary>
    private List<UsageEntry> SortItems(IEnumerable<UsageEntry> items)
    {
        Func<UsageEntry, IComparable> key = _sortCol switch
        {
            0 => e => e.Name,
            3 => e => e.Items,
            4 => e => e.Modified,
            5 => e => e.AttrText,
            _ => e => e.Bytes,      // 1 (% of Parent) and 2 (Size)
        };

        var cmp = _sortCol == 0 || _sortCol == 5
            ? Comparer<IComparable>.Create((a, b) => string.Compare((string)a, (string)b, StringComparison.OrdinalIgnoreCase))
            : Comparer<IComparable>.Default;

        var ordered = _sortDesc
            ? items.OrderByDescending(key, cmp)
            : items.OrderBy(key, cmp);

        return ordered.ThenByDescending(e => e.Bytes).ToList();
    }

    private void BuildDrives()
    {
        PnlDrives.Children.Clear();
        var drives = DiskUsageInfo.FixedDrives();
        foreach (var d in drives)
        {
            var btn = new Button
            {
                Content = d.Display,
                Tag     = d.Root,
                Style   = (Style)FindResource("ActionButton"),
                Margin  = new Thickness(0, 0, 8, 8),
                Padding = new Thickness(12, 6, 12, 6),
            };
            System.Windows.Automation.AutomationProperties.SetAutomationId(btn, $"Drive{d.Root[0]}");
            btn.Click += (_, _) => SelectDrive(d.Root);
            PnlDrives.Children.Add(btn);
        }
        if (drives.Count == 0)
            TxtStatus.Text = "No fixed drives found.";
        // One fixed drive (the common client-machine case) → scan it right away instead of making
        // the tech click the only chip there is. The MFT fast scan takes seconds and is read-only.
        else if (drives.Count == 1)
            SelectDrive(drives[0].Root);
    }

    /// <summary>Pick a drive: try the fast MFT scan first, then show it (or fall back to the walk).</summary>
    private async void SelectDrive(string root, string? goTo = null)
    {
        _cts?.Cancel();
        var cts = new CancellationTokenSource();
        _cts = cts;
        var ct = cts.Token;

        _mft = null; _mftRoot = null;
        _currentPath = root;
        TxtPath.Text = root;
        LstEntries.ItemsSource = null;
        Busy.Visibility = Visibility.Visible;
        TxtStatus.Text = $"⚡ Fast scan — reading the MFT for {root} …";

        MftVolume? vol = null;
        var sw = Stopwatch.StartNew();
        try { vol = await Task.Run(() => MftVolume.TryScan(root, ct), ct); }
        catch (OperationCanceledException) { return; }
        catch { }
        sw.Stop();

        if (_cts != cts) return;   // superseded
        if (vol != null) { _mft = vol; _mftRoot = root; }
        else TxtStatus.Text = "MFT unavailable (non-NTFS or blocked) — using the folder walk.";

        UpdateSummary(root, vol != null ? sw.Elapsed : null, vol != null);
        Navigate(goTo ?? root);
    }

    private async void Navigate(string path)
    {
        _currentPath = path;
        TxtPath.Text        = path;
        BtnRescan.IsEnabled = true;
        BtnOpen.IsEnabled   = true;

        // ── Fast path: in-memory MFT tree — instant, no walk. ──
        if (_mft != null && _mftRoot != null &&
            path.StartsWith(_mftRoot, StringComparison.OrdinalIgnoreCase))
        {
            _cts?.Cancel();   // stop any fallback walk still running
            _items = _mft.Children(path);
            _resultPrefix = "⚡";
            ApplyView();
            Busy.Visibility = Visibility.Collapsed;
            return;
        }

        // ── Fallback: recursive folder walk (two-phase: instant names, then sizes). ──
        _cts?.Cancel();
        var cts = new CancellationTokenSource();
        _cts = cts;
        var ct = cts.Token;
        Busy.Visibility = Visibility.Visible;
        LstEntries.ItemsSource = null;

        try
        {
            var quick = await Task.Run(() => DiskUsageInfo.QuickList(path), ct);
            if (_cts != cts) return;
            _items = quick; _resultPrefix = "";
            ApplyView();
            TxtStatus.Text = "Sizing folders… (you can click a folder to browse into it now)";

            var sized = await Task.Run(() => DiskUsageInfo.Children(path, ct), ct);
            if (ct.IsCancellationRequested || _cts != cts) return;
            _items = sized; _resultPrefix = "●";
            ApplyView();
        }
        catch (OperationCanceledException) { }
        catch (Exception ex) { TxtStatus.Text = "● Error: " + ex.Message; }
        finally { if (_cts == cts) Busy.Visibility = Visibility.Collapsed; }
    }

    // Build the visible list from _items: a ".." row (unless at a root) + children, filtering out
    // hidden items unless "Show hidden items" is ticked. Folder sizes are unaffected — the scan's
    // totals always include hidden content; this only hides rows.
    private void ApplyView()
    {
        if (_currentPath is null) return;
        bool showHidden = ChkHidden.IsChecked == true;

        var view = new List<UsageEntry>();
        if (!PathIsRoot(_currentPath))
        {
            var parent = Directory.GetParent(_currentPath.TrimEnd('\\'));
            if (parent != null)
                view.Add(new UsageEntry { Name = "..", FullPath = parent.FullName, IsDir = true, IsParent = true });
        }
        view.AddRange(SortItems(_items.Where(i => showHidden || !i.Hidden)));
        LstEntries.ItemsSource = view;

        if (_resultPrefix.Length > 0) SetResultStatus();
    }

    private void SetResultStatus()
    {
        bool showHidden = ChkHidden.IsChecked == true;
        long total = _items.Sum(i => i.Bytes);                     // folder total (always includes hidden)
        var vis = _items.Where(i => showHidden || !i.Hidden).ToList();
        int dirs = vis.Count(i => i.IsDir), files = vis.Count(i => !i.IsDir);
        int hidden = _items.Count(i => i.Hidden);
        string note = (!showHidden && hidden > 0) ? $"  ·  {hidden} hidden (tick to show)" : "";
        TxtStatus.Text = $"{_resultPrefix} {UsageEntry.FormatSize(total)} in {dirs} folder(s) + {files} file(s){note}  ·  double-click a folder to browse into it";
    }

    private void Hidden_Changed(object sender, RoutedEventArgs e) => ApplyView();

    // WizTree-style summary bar: total / used (%) / free (%) / scan mode + time.
    private void UpdateSummary(string root, TimeSpan? scanTime, bool fast)
    {
        try
        {
            var di = new DriveInfo(root[..2]);
            long total = di.TotalSize, free = di.AvailableFreeSpace, used = total - free;
            double usedPct = total > 0 ? used * 100.0 / total : 0;
            double freePct = total > 0 ? free * 100.0 / total : 0;
            string label = string.IsNullOrWhiteSpace(di.VolumeLabel) ? "Local Disk" : di.VolumeLabel;
            string mode  = fast ? "⚡ fast scan (MFT)" : "folder walk";
            string time  = scanTime is { } t ? $"   ·   scanned in {t.TotalSeconds:F2} s" : "";
            TxtSummary.Text =
                $"{root.TrimEnd('\\')} {label}   ·   Total {UsageEntry.FormatSize(total)}   ·   " +
                $"Used {UsageEntry.FormatSize(used)} ({usedPct:F1}%)   ·   Free {UsageEntry.FormatSize(free)} ({freePct:F1}%)   ·   {mode}{time}";
        }
        catch { TxtSummary.Text = ""; }
    }

    private static bool PathIsRoot(string path)
    {
        try
        {
            var full = Path.GetFullPath(path).TrimEnd('\\');
            var root = Path.GetPathRoot(path)?.TrimEnd('\\');
            return string.Equals(full, root, StringComparison.OrdinalIgnoreCase);
        }
        catch { return true; }
    }

    // Single left-click a folder (or the ".." row) to browse into it. Files do nothing.
    private void Row_Click(object sender, System.Windows.Input.MouseButtonEventArgs e)
    {
        if (sender is ListViewItem { DataContext: UsageEntry { IsDir: true } entry })
            Navigate(entry.FullPath);
    }

    private void Rescan_Click(object sender, RoutedEventArgs e)
    {
        if (_currentPath is null) return;
        // In fast mode, re-read the MFT (to pick up changes) then return to the current folder.
        if (_mftRoot != null) SelectDrive(_mftRoot, _currentPath);
        else Navigate(_currentPath);
    }

    private void Open_Click(object sender, RoutedEventArgs e)
    {
        if (_currentPath is null) return;
        try { Process.Start(new ProcessStartInfo(_currentPath) { UseShellExecute = true }); }
        catch (Exception ex)
        {
            MessageWindow.Show("Disk Usage", "Couldn't open the folder", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }
}
```
