---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ListWindow.xaml.cs
---

# PartnerTool\ListWindow.xaml.cs

```csharp
using System.Windows;

namespace PartnerTool;

/// <summary>One row in a <see cref="ListWindow"/>: a main line, a muted sub-line, and a right-aligned tag.</summary>
public record ListRow(string Main, string Sub, string Right);

/// <summary>A simple scrollable, titled list dialog — used for the full "see all" views
/// (e.g. complete Windows Update history, recent errors over the last 10 days).</summary>
public partial class ListWindow : Window
{
    public ListWindow(string title, IEnumerable<ListRow> rows)
    {
        InitializeComponent();
        Title = title;
        TxtTitle.Text = title;
        Rows.ItemsSource = rows.ToList();
    }

    private void Close_Click(object sender, RoutedEventArgs e) => Close();
}
```
