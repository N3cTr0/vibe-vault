---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BitLockerWindow.xaml.cs
---

# PartnerTool\BitLockerWindow.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Threading;

namespace PartnerTool;

public partial class BitLockerWindow : Window
{
    private readonly DispatcherTimer _copyTimer = new() { Interval = TimeSpan.FromSeconds(2) };

    public BitLockerWindow()
    {
        InitializeComponent();
        _copyTimer.Tick += (_, _) => { _copyTimer.Stop(); TxtCopied.Text = ""; };

        Loaded += async (_, _) =>
        {
            var keys = await Task.Run(BitLockerInfo.GetRecoveryKeys);
            KeyList.ItemsSource = keys;
            TxtStatus.Visibility = keys.Count == 0 ? Visibility.Visible : Visibility.Collapsed;
            if (keys.Count == 0)
                TxtStatus.Text = "No BitLocker recovery key is stored on this PC. The drive may be " +
                                 "unencrypted, or it uses TPM-only protection without a numerical recovery password.";
        };
    }

    private void Copy_Click(object sender, RoutedEventArgs e)
    {
        if (sender is Button { Tag: string pwd })
        {
            try
            {
                Clipboard.SetText(pwd);
                TxtCopied.Text = "Copied to clipboard.";
                _copyTimer.Stop();
                _copyTimer.Start();
            }
            catch { /* clipboard busy — ignore */ }
        }
    }

    private void Close_Click(object sender, RoutedEventArgs e) => Close();
}
```
