---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\MessageWindow.xaml.cs
---

# PartnerTool\MessageWindow.xaml.cs

```csharp
using System.Windows;
using System.Windows.Media;

namespace PartnerTool;

public enum MessageKind { Info, Warning, Error }

/// <summary>
/// A dark-themed replacement for the native MessageBox, matching the rest of the app.
/// Use the static <see cref="Show"/> / <see cref="Confirm"/> helpers.
/// </summary>
public partial class MessageWindow : Window
{
    private MessageWindow(string title, string heading, string body, MessageKind kind, bool yesNo)
    {
        InitializeComponent();
        Title           = title;
        TxtHeading.Text = heading;
        TxtBody.Text    = body;
        TxtHeading.Foreground = kind switch
        {
            MessageKind.Warning => new SolidColorBrush(Color.FromRgb(0xF9, 0xE2, 0xAF)),
            MessageKind.Error   => new SolidColorBrush(Color.FromRgb(0xF3, 0x8B, 0xA8)),
            _                   => new SolidColorBrush(Color.FromRgb(0xCD, 0xD6, 0xF4)),
        };

        if (yesNo)
        {
            BtnOk.Content        = "Yes";
            BtnCancel.Content    = "No";
            BtnCancel.Visibility = Visibility.Visible;
        }
    }

    private void Ok_Click(object sender, RoutedEventArgs e)     => DialogResult = true;
    private void Cancel_Click(object sender, RoutedEventArgs e) => DialogResult = false;

    /// <summary>Show a styled info/warning/error message with an OK button.</summary>
    public static void Show(string title, string heading, string body,
                            MessageKind kind = MessageKind.Info, Window? owner = null)
        => Build(title, heading, body, kind, yesNo: false, owner).ShowDialog();

    /// <summary>Show a styled Yes/No confirmation. Returns true for Yes.</summary>
    public static bool Confirm(string title, string heading, string body,
                               MessageKind kind = MessageKind.Warning, Window? owner = null)
        => Build(title, heading, body, kind, yesNo: true, owner).ShowDialog() == true;

    private static MessageWindow Build(string title, string heading, string body,
                                       MessageKind kind, bool yesNo, Window? owner)
    {
        var w = new MessageWindow(title, heading, body, kind, yesNo);
        if (owner != null)
        {
            w.Owner = owner;
            w.WindowStartupLocation = WindowStartupLocation.CenterOwner;
        }
        return w;
    }
}
```
