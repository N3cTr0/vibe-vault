---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SettingsWindow.xaml.cs
---

# PartnerTool\SettingsWindow.xaml.cs

```csharp
using System.IO;
using System.Windows;

namespace PartnerTool;

/// <summary>
/// Small settings dialog (opened from the sidebar). Edits the persisted options in settings.json:
/// log retention days (used by <see cref="LogRetention"/> at startup) and the hardware-sensor
/// kill-switch.
/// </summary>
public partial class SettingsWindow : Window
{
    public SettingsWindow()
    {
        InitializeComponent();
        var s = SettingsStore.Current;
        TxtRetentionDays.Text = s.LogRetentionDays.ToString();
        ChkSensors.IsChecked  = s.EnableSensors;
        ChkPreview.IsChecked  = s.ShowPreviewFeatures;
    }

    private void Save_Click(object sender, RoutedEventArgs e)
    {
        if (!int.TryParse(TxtRetentionDays.Text.Trim(), out int days) || days < 1 || days > 365)
        {
            MessageWindow.Show("Settings", "Invalid log retention",
                "Enter a number of days between 1 and 365.", MessageKind.Warning, this);
            return;
        }

        var s = SettingsStore.Current;
        s.LogRetentionDays     = days;
        s.EnableSensors        = ChkSensors.IsChecked == true;
        s.ShowPreviewFeatures  = ChkPreview.IsChecked == true;
        SettingsStore.Save();   // fires SettingsStore.Changed → MainWindow re-applies the preview tab

        ActivityLog.Action("Settings",
            $"Updated settings — log retention {days} day(s), sensors {(s.EnableSensors ? "on" : "off")}, " +
            $"preview features {(s.ShowPreviewFeatures ? "on" : "off")}");
        DialogResult = true;
    }

    /// <summary>
    /// Delete every log file in C:\PCI\Logs right now, regardless of retention age. Tech-gated —
    /// it erases the audit trail — and the purge itself is recorded as the first entry of the
    /// fresh activity log so the trail shows it happened.
    /// </summary>
    private void PurgeLogs_Click(object sender, RoutedEventArgs e)
    {
        if (!TechGate.Verify(this)) return;

        int count = 0; long bytes = 0; int locked = 0;
        try
        {
            var dir = new DirectoryInfo(@"C:\PCI\Logs");
            if (dir.Exists)
            {
                foreach (var f in dir.GetFiles())
                {
                    try { long len = f.Length; f.Delete(); count += 1; bytes += len; }
                    catch { locked += 1; }   // in use — leave it; retention gets it later
                }
            }
        }
        catch (Exception ex)
        {
            TxtPurgeStatus.Text = $"Purge failed: {ex.Message}";
            return;
        }

        ActivityLog.Action("Logs", $"Purged all log files ({count} file(s), {bytes / 1048576.0:F1} MB)"
                                   + (locked > 0 ? $" — {locked} in use, skipped" : ""));
        TxtPurgeStatus.Text = $"Deleted {count} file(s) ({bytes / 1048576.0:F1} MB)"
                              + (locked > 0 ? $"; {locked} in use, skipped." : ".");
    }

    private void Cancel_Click(object sender, RoutedEventArgs e) => DialogResult = false;
}
```
