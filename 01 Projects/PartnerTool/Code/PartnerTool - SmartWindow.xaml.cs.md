---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SmartWindow.xaml.cs
---

# PartnerTool\SmartWindow.xaml.cs

```csharp
using System.Windows;
using System.Windows.Media;

namespace PartnerTool;

public partial class SmartWindow : Window
{
    // View-model wrappers so the XAML can bind formatted text + severity colours.
    private sealed record AttrRow(SmartAttr A)
    {
        public string IdText     => A.Id.ToString();
        public string Name       => A.Name;
        public string Current    => A.Current.ToString();
        public string Worst      => A.Worst.ToString();
        public string ThreshText => A.Threshold is { } t && t > 0 ? t.ToString() : "—";
        public string RawText    => A.Raw.ToString("N0");
        // A value at/under its threshold is a failing attribute — flag it red.
        public Brush  RowBrush   => A.BelowThreshold ? StatusColors.Red : StatusColors.Text;
    }

    private sealed record DriveCard(SmartDrive D)
    {
        public string Title => CleanInstance(D.Instance);
        public IReadOnlyList<AttrRow> Attributes => D.Attributes.Select(a => new AttrRow(a)).ToList();

        public string StatusText => D.PredictFailure switch
        {
            true  => "⚠ FAILURE PREDICTED",
            false => "● OK",
            _     => "status unknown",
        };
        public Brush StatusBrush => D.PredictFailure switch
        {
            true  => StatusColors.Red,
            false => StatusColors.Green,
            _     => StatusColors.Muted,
        };

        // Instance names look like "IDE\\DiskSAMSUNG…\\5&…" — keep the readable middle chunk.
        private static string CleanInstance(string inst)
        {
            if (string.IsNullOrWhiteSpace(inst)) return "Drive";
            var parts = inst.Split('\\');
            var mid = parts.Length > 1 ? parts[1] : inst;
            return mid.StartsWith("Disk", StringComparison.OrdinalIgnoreCase) ? mid[4..] : mid;
        }
    }

    public SmartWindow(List<SmartDrive> drives)
    {
        InitializeComponent();
        if (drives.Count == 0)
        {
            TxtEmpty.Visibility = Visibility.Visible;
        }
        else
        {
            IcDrives.ItemsSource = drives.Select(d => new DriveCard(d)).ToList();
        }
    }

    private void Close_Click(object sender, RoutedEventArgs e) => Close();
}
```
