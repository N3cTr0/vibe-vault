---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PowerPlanWindow.xaml.cs
---

# PartnerTool\PowerPlanWindow.xaml.cs

```csharp
using System.Windows;

namespace PartnerTool;

public partial class PowerPlanWindow : Window
{
    public PowerPlanWindow()
    {
        InitializeComponent();
        Loaded += (_, _) => Refresh();
    }

    public record PlanVM(string Guid, string Name, Visibility ActiveVis, Visibility InactiveVis);

    private void Refresh()
    {
        var opts = PowerMode.List();
        PlanList.ItemsSource = opts.Select(o => new PlanVM(
            o.Guid.ToString(), o.Name,
            o.Active ? Visibility.Visible : Visibility.Collapsed,
            o.Active ? Visibility.Collapsed : Visibility.Visible)).ToList();
        TxtEmpty.Visibility = PowerMode.Supported() ? Visibility.Collapsed : Visibility.Visible;
    }

    private void SetActive_Click(object sender, RoutedEventArgs e)
    {
        if (sender is FrameworkElement { Tag: string guidStr } && Guid.TryParse(guidStr, out var g))
        {
            PowerMode.Set(g);
            Refresh();
        }
    }

    private void Close_Click(object sender, RoutedEventArgs e) => Close();
}
```
