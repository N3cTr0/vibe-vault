---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\MainWindow.xaml
---

# src\Octavia.App\MainWindow.xaml

```xml
<Window x:Class="Octavia.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:wv2="clr-namespace:Microsoft.Web.WebView2.Wpf;assembly=Microsoft.Web.WebView2.Wpf"
        Title="Octavia"
        Height="780" Width="1100"
        MinHeight="520" MinWidth="720"
        Background="#D4D0C4"
        WindowStartupLocation="CenterScreen">
    <Grid>
        <wv2:WebView2 x:Name="Face" DefaultBackgroundColor="#D4D0C4" />
        <TextBlock x:Name="Fallback"
                   Visibility="Collapsed"
                   Margin="48"
                   TextWrapping="Wrap"
                   FontFamily="Segoe UI"
                   FontSize="15"
                   Foreground="#242219"
                   VerticalAlignment="Center"
                   HorizontalAlignment="Center"
                   TextAlignment="Center" />
    </Grid>
</Window>
```
