---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\HealthCheckPage.xaml.cs
---

# PartnerTool\Pages\HealthCheckPage.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media;
using System.Windows.Media.Animation;

namespace PartnerTool.Pages;

public partial class HealthCheckPage : UserControl
{
    private static readonly SolidColorBrush DimRing = Freeze("#313244");
    private bool _busy;
    private List<HealthFinding> _findings = new();

    public HealthCheckPage() => InitializeComponent();

    /// <summary>One category block in the two-column findings layout.</summary>
    public sealed class FindingGroup
    {
        public required string Name { get; init; }
        public required List<HealthFinding> Items { get; init; }
        public string Summary => Items.Count(f => f.Severity != HealthSeverity.Good) is var n && n > 0
            ? $"{n} issue(s)" : "all clear";
    }

    // Fixed display order — problem areas a tech acts on first, OEM-specific last. Categories
    // not listed (future additions) fall to the end alphabetically.
    private static readonly string[] GroupOrder =
        ["Security", "Stability", "Disk", "Updates", "Junk", "Startup", "Shortcuts", "Maintenance", "Dell"];

    /// <summary>
    /// Group findings by category and split the groups across the two columns, keeping the
    /// display order and balancing by row count so the columns end up roughly equal height.
    /// </summary>
    private void ShowFindings()
    {
        var groups = _findings
            .GroupBy(f => f.Category)
            .Select(g => new FindingGroup { Name = g.Key.ToUpperInvariant(), Items = g.ToList() })
            .OrderBy(g => { int i = Array.IndexOf(GroupOrder, g.Items[0].Category); return i < 0 ? int.MaxValue : i; })
            .ThenBy(g => g.Name)
            .ToList();

        var left  = new List<FindingGroup>();
        var right = new List<FindingGroup>();
        int lRows = 0, rRows = 0;
        foreach (var g in groups)
        {
            // +1 counts the group header itself so many small groups don't all pile up one side.
            if (lRows <= rRows) { left.Add(g);  lRows += g.Items.Count + 1; }
            else                { right.Add(g); rRows += g.Items.Count + 1; }
        }
        IcGroupsL.ItemsSource = left;
        IcGroupsR.ItemsSource = right;
    }

    // ── Scan (the ring is the button) ─────────────────────────────────────
    private async void Ring_Click(object sender, System.Windows.Input.MouseButtonEventArgs? e)
    {
        if (_busy) return;
        _busy = true;

        // Clear the previous scan so old findings don't linger while the new scan runs.
        _findings = new();
        IcGroupsL.ItemsSource = null;
        IcGroupsR.ItemsSource = null;
        TxtNoFindings.Visibility = Visibility.Collapsed;
        BtnFixSelected.IsEnabled = false;
        BtnFixSelected.Content = "Fix Selected";   // reset from a previous "Nothing to Fix"
        FixLogPanel.Visibility = Visibility.Collapsed;

        // Ring → "scanning" state: dim base + spinning bright arc, progress bar, live step text.
        Ring.Cursor = System.Windows.Input.Cursors.Wait;
        EllRing.Stroke = DimRing;
        TxtScore.FontSize = 26;
        TxtScore.Text = "0%"; TxtScore.Foreground = StatusColors.Blue;
        TxtScoreSub.Text = "scanning";
        TxtStatus.Text = "Starting scan…";
        ProgBar.Value = 0; ProgBar.Visibility = Visibility.Visible;
        StartSpin();

        try
        {
            var report = await HealthCheck.RunAsync((pct, msg) =>
            {
                // Runs on the UI thread (RunAsync doesn't ConfigureAwait(false)).
                ProgBar.Value = pct;
                TxtScore.Text = $"{pct}%";
                TxtStatus.Text = msg;
            });

            _findings = report.Findings;
            ShowFindings();
            TxtNoFindings.Visibility = _findings.Count == 0 ? Visibility.Visible : Visibility.Collapsed;

            var color = report.Score >= 85 ? StatusColors.Green
                      : report.Score >= 60 ? StatusColors.Yellow
                      : StatusColors.Red;
            EllRing.Stroke = color;
            TxtScore.FontSize = 44;
            TxtScore.Text = report.Score.ToString(); TxtScore.Foreground = color;
            TxtScoreSub.Text = "/ 100";
            TxtStatus.Text = report.ProblemCount == 0
                ? "All clear — no issues found. Click the circle to rescan."
                : $"{report.ProblemCount} issue(s) found — click the circle to rescan.";

            UpdateFixButton();
        }
        catch (Exception ex)
        {
            EllRing.Stroke = StatusColors.Red;
            TxtScore.FontSize = 26; TxtScore.Text = "—"; TxtScoreSub.Text = "error";
            TxtStatus.Text = $"Scan failed: {ex.Message}";
        }
        finally
        {
            StopSpin();
            ProgBar.Visibility = Visibility.Collapsed;
            Ring.Cursor = System.Windows.Input.Cursors.Hand;
            _busy = false;
        }
    }

    private void StartSpin()
    {
        SpinArc.Visibility = Visibility.Visible;
        SpinRot.BeginAnimation(RotateTransform.AngleProperty,
            new DoubleAnimation(0, 360, new Duration(TimeSpan.FromSeconds(1.1)))
            { RepeatBehavior = RepeatBehavior.Forever });
    }

    private void StopSpin()
    {
        SpinRot.BeginAnimation(RotateTransform.AngleProperty, null);
        SpinArc.Visibility = Visibility.Collapsed;
    }

    private static SolidColorBrush Freeze(string hex)
    {
        var b = (SolidColorBrush)new BrushConverter().ConvertFromString(hex)!;
        b.Freeze();
        return b;
    }

    private void UpdateFixButton()
    {
        int selectable = _findings.Count(f => f.IsFixable);
        BtnFixSelected.IsEnabled = selectable > 0;
        BtnFixSelected.Content = selectable > 0 ? "Fix Selected" : "Nothing to Fix";
    }

    // ── Advisory "Open →" → jump to the relevant tab ──────────────────────
    private void Open_Click(object sender, RoutedEventArgs e)
    {
        if (sender is Button { Tag: string tag } && Window.GetWindow(this) is MainWindow mw)
            mw.NavigateTo(tag);
    }

    // ── Fix Selected — run the checked in-place fixes ─────────────────────
    private async void FixSelected_Click(object sender, RoutedEventArgs e)
    {
        if (_busy) return;
        var selected = _findings.Where(f => f.IsFixable && f.Selected).ToList();
        if (selected.Count == 0)
        {
            MessageWindow.Show("Health Check", "Nothing selected",
                "Tick the items you want to fix, then click Fix Selected.", MessageKind.Info, Window.GetWindow(this));
            return;
        }

        // One tech-gate for the whole batch if any selected fix deletes files.
        if (selected.Any(f => f.NeedsGate) && !TechGate.Verify(Window.GetWindow(this))) return;

        _busy = true;
        BtnFixSelected.IsEnabled = false;
        FixLogPanel.Visibility = Visibility.Visible;
        TxtFixLog.Text = "";
        void Log(string s) => Dispatcher.Invoke(() =>
        {
            TxtFixLog.Text += s + "\n";
            FixLogScroll.ScrollToBottom();
        });

        ActivityLog.Action("Health Check", $"Fix selected: {string.Join(", ", selected.Select(s => s.Title))}");
        try
        {
            foreach (var f in selected)
            {
                Log($"▶ {f.Title}");
                try
                {
                    var result = await f.Fixer!(Log);
                    Log($"  ✓ {result}");
                    ActivityLog.Result("Health Check", $"{f.Title}: {result}");
                }
                catch (Exception ex)
                {
                    Log($"  ✗ {ex.Message}");
                    ActivityLog.Result("Health Check", $"{f.Title}: failed — {ex.Message}");
                }
            }
            Log("━━━ Done — rescanning ━━━");
        }
        finally { _busy = false; }

        // Rescan so fixed items turn green and the score updates.
        Ring_Click(this, null!);
    }
}
```
