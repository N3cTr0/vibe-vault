---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TechGateWindow.xaml.cs
---

# PartnerTool\TechGateWindow.xaml.cs

```csharp
using System.Windows;

namespace PartnerTool;

public partial class TechGateWindow : Window
{
    /// <param name="expired">
    /// True when the session was unlocked but has since idled past <see cref="TechGate.Timeout"/>.
    /// Say so, otherwise a tech who verified 30 minutes ago thinks the gate is broken.
    /// </param>
    public TechGateWindow(bool expired = false)
    {
        InitializeComponent();
        if (expired)
            TxtPrompt.Text = $"Tech verification expired after {TechGate.Timeout.TotalMinutes:0} minutes idle.\n" +
                             "Enter tech code to continue.";
        Loaded += (_, _) => TxtCode.Focus();
    }

    private void Ok_Click(object sender, RoutedEventArgs e)
    {
        if (TxtCode.Password.Trim() == TechGate.TodayCode)
        {
            DialogResult = true;
            Close();
        }
        else
        {
            TxtErr.Text = "Incorrect code. Ask the tech lead if you're not sure.";
            TxtErr.Visibility = Visibility.Visible;
            TxtCode.SelectAll();
            TxtCode.Focus();
        }
    }

    private void Cancel_Click(object sender, RoutedEventArgs e)
    {
        DialogResult = false;
        Close();
    }
}
```
