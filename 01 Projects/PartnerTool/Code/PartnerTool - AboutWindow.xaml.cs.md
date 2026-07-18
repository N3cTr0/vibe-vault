---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\AboutWindow.xaml.cs
---

# PartnerTool\AboutWindow.xaml.cs

```csharp
using System.Diagnostics;
using System.Reflection;
using System.Windows;
using System.Windows.Documents;
using System.Windows.Navigation;
using System.Windows.Threading;

namespace PartnerTool;

public partial class AboutWindow : Window
{
    private readonly DispatcherTimer _copyTimer = new() { Interval = TimeSpan.FromSeconds(1.1) };

    public AboutWindow()
    {
        InitializeComponent();

        _copyTimer.Tick += (_, _) => { _copyTimer.Stop(); CopiedPopup.IsOpen = false; };

        var v = Assembly.GetExecutingAssembly().GetName().Version;
        TxtVersion.Text = v != null
            ? $"Version {v.Major}.{v.Minor}.{v.Build}"
            : "Version 1.0.0";
    }

    private void Link_RequestNavigate(object sender, RequestNavigateEventArgs e)
    {
        try { Process.Start(new ProcessStartInfo(e.Uri.AbsoluteUri) { UseShellExecute = true }); }
        catch { /* no browser / blocked — ignore */ }
        e.Handled = true;
    }

    private void Phone_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Hyperlink h || h.Tag is not string number) return;

        try { Clipboard.SetText(number); } catch { /* clipboard busy — ignore */ }

        // Re-open so Placement=Mouse repositions the toast at the current cursor.
        CopiedPopup.IsOpen = false;
        CopiedPopup.IsOpen = true;
        _copyTimer.Stop();
        _copyTimer.Start();
    }

    private void Ok_Click(object sender, RoutedEventArgs e) => Close();
}
```
