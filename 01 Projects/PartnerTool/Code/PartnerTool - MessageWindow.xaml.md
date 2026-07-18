---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\MessageWindow.xaml
---

# PartnerTool\MessageWindow.xaml

```xml
<Window x:Class="PartnerTool.MessageWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Width="460" SizeToContent="Height" MinWidth="380"
        ResizeMode="NoResize" WindowStartupLocation="CenterScreen"
        ShowInTaskbar="False" Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <TextBlock x:Name="TxtHeading" Grid.Row="0" FontSize="15" FontWeight="SemiBold"
                   Foreground="#CDD6F4" TextWrapping="Wrap"/>

        <TextBlock x:Name="TxtBody" Grid.Row="1" FontSize="12" Foreground="#BAC2DE"
                   TextWrapping="Wrap" Margin="0,10,0,0"/>

        <StackPanel Grid.Row="2" Orientation="Horizontal" HorizontalAlignment="Right" Margin="0,20,0,0">
            <Button x:Name="BtnCancel" Content="No" Style="{StaticResource ActionButton}"
                    MinWidth="80" Margin="0,0,8,0" Visibility="Collapsed"
                    IsCancel="True" Click="Cancel_Click"/>
            <Button x:Name="BtnOk" Content="OK" Style="{StaticResource ActionButton}"
                    MinWidth="80" IsDefault="True" Click="Ok_Click"/>
        </StackPanel>
    </Grid>
</Window>
```
