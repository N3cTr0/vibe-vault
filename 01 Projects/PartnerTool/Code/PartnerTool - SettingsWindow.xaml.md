---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SettingsWindow.xaml
---

# PartnerTool\SettingsWindow.xaml

```xml
<Window x:Class="PartnerTool.SettingsWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Settings"
        Width="480" SizeToContent="Height"
        ResizeMode="NoResize" WindowStartupLocation="CenterOwner"
        Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <StackPanel Grid.Row="0">
            <TextBlock Text="Settings" FontSize="16" FontWeight="SemiBold" Foreground="#CDD6F4"/>
            <TextBlock Text="Saved to settings.json next to the exe — applies to this machine."
                       Foreground="#6C7086" FontSize="11" Margin="0,6,0,0" TextWrapping="Wrap"/>
        </StackPanel>

        <StackPanel Grid.Row="1" Margin="0,16,0,0">

            <!-- Log retention -->
            <TextBlock Text="LOG RETENTION" Foreground="#CBA6F7" FontSize="11" FontWeight="SemiBold"/>
            <DockPanel Margin="0,8,0,4">
                <Border DockPanel.Dock="Right" Background="#45475A" CornerRadius="6" Padding="10,0">
                    <TextBox x:Name="TxtRetentionDays" Width="56" Height="30" Background="Transparent"
                             Foreground="#CDD6F4" FontSize="12" BorderThickness="0"
                             VerticalContentAlignment="Center" TextAlignment="Center"/>
                </Border>
                <TextBlock Text="Delete logs in C:\PCI\Logs older than (days)" Foreground="#CDD6F4"
                           FontSize="12" VerticalAlignment="Center" Margin="0,0,10,0"/>
            </DockPanel>
            <TextBlock Text="Runs at every startup. 1–365 days; default 30. The activity log survives as long as the tool is in regular use."
                       Foreground="#6C7086" FontSize="10" TextWrapping="Wrap"/>
            <DockPanel Margin="0,10,0,0">
                <Button DockPanel.Dock="Right" x:Name="BtnPurgeLogs" Content="Purge Now"
                        Style="{StaticResource ActionButton}" Padding="14,5" Click="PurgeLogs_Click"/>
                <TextBlock x:Name="TxtPurgeStatus" Foreground="#6C7086" FontSize="10" TextWrapping="Wrap"
                           VerticalAlignment="Center" Margin="0,0,10,0"
                           Text="Delete ALL current log files in C:\PCI\Logs, regardless of age."/>
            </DockPanel>

        </StackPanel>

        <StackPanel Grid.Row="2" Orientation="Horizontal" HorizontalAlignment="Right" Margin="0,18,0,0">
            <Button Content="Cancel" Style="{StaticResource ActionButton}" IsCancel="True"
                    Margin="0,0,8,0" Click="Cancel_Click"/>
            <Button Content="Save" Style="{StaticResource ActionButton}" IsDefault="True"
                    Click="Save_Click"/>
        </StackPanel>
    </Grid>
</Window>
```
