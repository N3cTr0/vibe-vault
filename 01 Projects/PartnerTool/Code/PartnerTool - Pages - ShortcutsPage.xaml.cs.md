---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ShortcutsPage.xaml.cs
---

# PartnerTool\Pages\ShortcutsPage.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Windows;
using System.Windows.Controls;

namespace PartnerTool.Pages;

public partial class ShortcutsPage : UserControl
{
    // Print Management is an optional Windows feature; the console lives in System32 when installed.
    private static string PrintMgmtPath =>
        Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.System), "printmanagement.msc");
    private const string PrintMgmtCapability = "Print.Management.Console~~~~0.0.1.0";
    private bool _printBusy;

    public ShortcutsPage() => InitializeComponent();

    private void Tool_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: string tag }) return;
        try
        {
            // Security: this app is elevated and ShellExecute searches the working directory for
            // bare names — always pin tools to System32/Windows so a planted file can't run as admin.
            if (tag.StartsWith("rundll32://"))
            {
                var args = tag["rundll32://".Length..];
                int comma = args.IndexOf(',');
                if (comma > 0) args = ProcessRunner.ResolveSystemTool(args[..comma]) + args[comma..];   // pin the .cpl too
                Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemTool("rundll32.exe"), args) { UseShellExecute = true });
            }
            else if (tag.StartsWith("explorer://"))
            {
                // Open a shell namespace folder (e.g. classic Devices and Printers) via Explorer.
                var explorer = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Windows), "explorer.exe");
                Process.Start(new ProcessStartInfo(explorer, tag["explorer://".Length..]) { UseShellExecute = true });
            }
            else
                Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemTool(tag)) { UseShellExecute = true });
        }
        catch (Exception ex)
        {
            MessageWindow.Show("Error", "Couldn't open", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }

    private async void PrintMgmt_Click(object sender, RoutedEventArgs e)
    {
        if (_printBusy) return;

        if (File.Exists(PrintMgmtPath)) { LaunchPrintMgmt(); return; }

        if (!MessageWindow.Confirm("Print Management",
                "Print Management isn't installed on this PC.",
                "It's an optional Windows feature. Install it now? This downloads from Windows Update and may take a minute or two.",
                MessageKind.Info, Window.GetWindow(this)))
            return;

        _printBusy = true;
        BtnPrintMgmt.IsEnabled        = false;
        PrintProgress.Visibility      = Visibility.Visible;
        PrintProgress.IsIndeterminate = true;          // until DISM reports a %
        PrintProgress.Value           = 0;
        TxtPrintStatus.Visibility     = Visibility.Visible;
        TxtPrintStatus.Foreground     = StatusColors.Yellow;
        TxtPrintStatus.Text           = "Installing Print Management…";
        ActivityLog.Action("Shortcuts", "Install Print Management capability (DISM)");

        int code = await ProcessRunner.RunAsync("dism.exe",
            $"/Online /Add-Capability /CapabilityName:{PrintMgmtCapability} /NoRestart",
            null,
            line => Dispatcher.Invoke(() => TxtPrintStatus.Text = line),
            pct  => Dispatcher.Invoke(() =>
            {
                PrintProgress.IsIndeterminate = false;
                PrintProgress.Value = pct;
                TxtPrintStatus.Text = $"Installing Print Management… {pct}%";
            }));

        PrintProgress.IsIndeterminate = false;
        if (code == 0 && File.Exists(PrintMgmtPath))
        {
            PrintProgress.Value       = 100;
            TxtPrintStatus.Text       = "● Installed";
            TxtPrintStatus.Foreground = StatusColors.Green;
            LaunchPrintMgmt();
        }
        else
        {
            TxtPrintStatus.Text       = $"● Install failed (DISM exit {code}). Try Settings → System → Optional features.";
            TxtPrintStatus.Foreground = StatusColors.Red;
        }
        BtnPrintMgmt.IsEnabled = true;
        _printBusy = false;
    }

    private void LaunchPrintMgmt()
    {
        try { Process.Start(new ProcessStartInfo(PrintMgmtPath) { UseShellExecute = true }); }
        catch (Exception ex)
        {
            MessageWindow.Show("Error", "Couldn't open Print Management", ex.Message,
                MessageKind.Error, Window.GetWindow(this));
        }
    }
}
```
