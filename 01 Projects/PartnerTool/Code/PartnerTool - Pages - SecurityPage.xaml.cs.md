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

    // Jump to the Windows setting/applet that changes this audit item - or, for the special
    // reset-execution-policy target, do the reset in-app after confirming.
    private async void Fix_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Hyperlink { DataContext: AuditItem { Fix: { } fix } }) return;

        if (fix.Target == SecurityAudit.ResetExecPolicyTarget)
        {
            if (!TechGate.Verify(Window.GetWindow(this))) return;
            if (!MessageWindow.Confirm("PowerShell execution policy",
                    "Reset to the Windows default?",
                    "This clears the machine PowerShell execution policy (currently loosened) so it " +
                    "reverts to Windows' default. Scripts that relied on Bypass/Unrestricted may stop " +
                    "running until re-signed or re-allowed. Continue?",
                    MessageKind.Warning, Window.GetWindow(this)))
                return;
            ActivityLog.Action("Security", "Reset PowerShell execution policy to Windows default");
            await Task.Run(SecurityAudit.ResetExecutionPolicy);
            IcAudit.ItemsSource = await Task.Run(SecurityAudit.Collect);   // reflect the change
            return;
        }

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
                // System32 tool/applet - launch by absolute path (we run elevated).
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

    private void OpenDefender_Click(object sender, RoutedEventArgs e)
        => OpenWindowsSecurity("windowsdefender://", "Windows Security");

    // windowsdefender://threat lands on Virus & threat protection, which carries Current threats and
    // a Protection history link. There is no working URI for the history page itself - the obvious
    // windowsdefender://protectionhistory is not recognised and silently drops you on Home.
    private void OpenThreatHistory_Click(object sender, RoutedEventArgs e)
        => OpenWindowsSecurity("windowsdefender://threat", "Virus & threat protection");

    private void OpenWindowsSecurity(string uri, string what)
    {
        ActivityLog.Action("Security", $"Open {what}");
        try
        {
            Process.Start(new ProcessStartInfo(uri) { UseShellExecute = true });
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Windows Security", $"Couldn't open {what}",
                "Windows wouldn't open the Security app on this machine.\n\n" + ex.Message,
                MessageKind.Warning, Window.GetWindow(this));
        }
    }

    private static async Task<(T value, long ms)> Timed<T>(Func<T> work)
    {
        var sw = Stopwatch.StartNew();
        var v  = await Task.Run(work);
        return (v, sw.ElapsedMilliseconds);
    }

    // Every card paints the moment its own collector returns, rather than all four waiting on the
    // slowest. Defender's and BitLocker's WMI providers can each take many seconds on a managed
    // machine, and one of them shouldn't leave the whole page blank. The per-collector timings go
    // to the activity log so a slow machine says which one it was.
    private async Task LoadAsync()
    {
        var auditTask = Timed(SecurityAudit.Collect);
        var defTask   = Timed(DefenderInfo.Collect);
        var proTask   = Timed(ProsentryInfo.Collect);
        var blTask    = Timed(BitLockerInfo.AnyProtectedVolume);

        long tAudit = 0, tDef = 0, tPro = 0, tBl = 0;

        async Task PaintAudit()
        {
            var (items, ms) = await auditTask;
            IcAudit.ItemsSource = items; tAudit = ms;
        }
        async Task PaintDef()
        {
            var (info, ms) = await defTask;
            PaintDefender(info); tDef = ms;
        }
        async Task PaintPro()
        {
            var (pro, ms) = await proTask;
            IcProsentry.ItemsSource  = pro.Tools;
            IcManagement.ItemsSource = new[] { pro.Intune };
            tPro = ms;
        }
        async Task PaintBitLocker()
        {
            var (any, ms) = await blTask;
            BitLockerCard.Visibility = any ? Visibility.Visible : Visibility.Collapsed;
            tBl = ms;
        }

        await Task.WhenAll(PaintAudit(), PaintDef(), PaintPro(), PaintBitLocker());

        ActivityLog.Result("Security",
            $"page load: audit {tAudit} ms, defender {tDef} ms, prosentry {tPro} ms, bitlocker {tBl} ms");
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
        TxtSigDate.Text    = d.SignatureUpdated is { } s ? s.ToString(Dates.DateTime) : "-";
        if (d.SignatureUpdated is { } su)
            TxtSigDate.Foreground = (DateTime.Now - su).TotalDays > 7 ? StatusColors.Yellow : StatusColors.Green;
        TxtQuick.Text      = d.QuickScanAgeDays is { } qa ? $"{qa} day(s) ago" : "-";
        TxtFull.Text       = d.FullScanAgeDays is { } fa ? $"{fa} day(s) ago" : "Never / unknown";
        // A non-zero count becomes a link straight to Protection history so the tech can see what
        // was found instead of hunting through Windows Security.
        TxtThreats.Inlines.Clear();
        if (d.ThreatCount == 0)
        {
            TxtThreats.Inlines.Add(new System.Windows.Documents.Run("None recorded"));
            TxtThreats.Foreground = StatusColors.Green;
        }
        else
        {
            var link = new System.Windows.Documents.Hyperlink(
                new System.Windows.Documents.Run($"{d.ThreatCount} in history"))
            {
                Foreground = StatusColors.Yellow,
                ToolTip    = "Open Protection history in Windows Security",
            };
            link.Click += OpenThreatHistory_Click;
            TxtThreats.Inlines.Add(link);
        }
    }
}
```
