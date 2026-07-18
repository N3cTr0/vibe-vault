---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SecurityPage.xaml.cs
---

# PartnerTool\Pages\SecurityPage.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Documents;

namespace PartnerTool.Pages;

public partial class SecurityPage : UserControl
{
    private bool _loadedOnce;

    public SecurityPage()
    {
        InitializeComponent();
        IsVisibleChanged += async (_, _) =>
        {
            if (IsVisible && !_loadedOnce) { _loadedOnce = true; await LoadAsync(); }
        };
    }

    private void BitLocker_Click(object sender, RoutedEventArgs e)
    {
        ActivityLog.Action("Security", "Open BitLocker recovery key viewer");
        new BitLockerWindow { Owner = Window.GetWindow(this) }.ShowDialog();
    }

    // Jump to the Windows setting/applet that changes this audit item.
    private void Fix_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Hyperlink { DataContext: AuditItem { Fix: { } fix } }) return;
        ActivityLog.Action("Security", $"Open Windows setting: {fix.Tooltip}");
        try
        {
            ProcessStartInfo psi;
            if (fix.ShellExecute)
            {
                // ms-settings: URI, or an .msc console (pin .msc to System32 so a
                // planted file in a writable working dir can't be opened instead).
                var target = fix.Target;
                if (target.EndsWith(".msc", StringComparison.OrdinalIgnoreCase) && !target.Contains('\\'))
                    target = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.System), target);
                psi = new ProcessStartInfo(target, fix.Args ?? "") { UseShellExecute = true };
            }
            else
            {
                // System32 tool/applet — launch by absolute path (we run elevated).
                psi = new ProcessStartInfo(ProcessRunner.ResolveSystemExe(fix.Target), fix.Args ?? "")
                { UseShellExecute = false };
            }
            Process.Start(psi);
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Couldn't open setting", fix.Tooltip,
                "Windows wouldn't open that settings page on this machine.\n\n" + ex.Message,
                MessageKind.Warning, Window.GetWindow(this));
        }
    }

    private async Task LoadAsync()
    {
        var auditTask = Task.Run(SecurityAudit.Collect);
        var defTask   = Task.Run(DefenderInfo.Collect);
        await Task.WhenAll(auditTask, defTask);

        IcAudit.ItemsSource = await auditTask;
        PaintDefender(await defTask);
    }

    private void PaintDefender(DefenderInfo d)
    {
        TxtNoDefender.Visibility = d.Available ? Visibility.Collapsed : Visibility.Visible;
        DefenderGrid.Visibility  = d.Available ? Visibility.Visible : Visibility.Collapsed;
        if (!d.Available) return;

        TxtRtp.Text        = d.RealTimeProtection ? "● On" : "● Off";
        TxtRtp.Foreground  = d.RealTimeProtection ? StatusColors.Green : StatusColors.Red;
        TxtTamper.Text     = d.TamperProtection ? "On" : "Off";
        TxtSig.Text        = d.SignatureVersion;
        TxtSigDate.Text    = d.SignatureUpdated is { } s ? s.ToString("d MMM yyyy HH:mm") : "—";
        if (d.SignatureUpdated is { } su)
            TxtSigDate.Foreground = (DateTime.Now - su).TotalDays > 7 ? StatusColors.Yellow : StatusColors.Green;
        TxtQuick.Text      = d.QuickScanAgeDays is { } qa ? $"{qa} day(s) ago" : "—";
        TxtFull.Text       = d.FullScanAgeDays is { } fa ? $"{fa} day(s) ago" : "Never / unknown";
        TxtThreats.Text    = d.ThreatCount == 0 ? "None recorded" : $"{d.ThreatCount} in history";
        TxtThreats.Foreground = d.ThreatCount == 0 ? StatusColors.Green : StatusColors.Yellow;
    }
}
```
