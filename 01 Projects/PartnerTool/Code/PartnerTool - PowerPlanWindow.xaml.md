---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PowerPlanWindow.xaml
---

# PartnerTool\PowerPlanWindow.xaml

```xml
<Window x:Class="PartnerTool.PowerPlanWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Power Mode"
        Width="440" SizeToContent="Height" MaxHeight="520"
        ResizeMode="NoResize" WindowStartupLocation="CenterOwner"
        Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <StackPanel Grid.Row="0">
            <TextBlock Text="Power Mode" FontSize="16" FontWeight="SemiBold" Foreground="#CDD6F4"/>
            <TextBlock Text="Optimize this machine for battery life or performance (the Windows 'Power Mode' setting)."
                       Foreground="#6C7086" FontSize="11" Margin="0,6,0,0" TextWrapping="Wrap"/>
        </StackPanel>

        <StackPanel Grid.Row="1" Margin="0,14,0,0">
            <ItemsControl x:Name="PlanList">
                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <Border Background="#313244" CornerRadius="8" Padding="14,12" Margin="0,0,0,8">
                            <DockPanel>
                                <TextBlock DockPanel.Dock="Right" Text="● Active" Foreground="#A6E3A1" FontSize="12"
                                           VerticalAlignment="Center" Visibility="{Binding ActiveVis}"/>
                                <Button DockPanel.Dock="Right" Content="Set" Tag="{Binding Guid}"
                                        Style="{StaticResource ActionButton}" Padding="14,5" FontSize="11"
                                        Click="SetActive_Click" Visibility="{Binding InactiveVis}"/>
                                <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="13" VerticalAlignment="Center"/>
                            </DockPanel>
                        </Border>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>
            <TextBlock x:Name="TxtEmpty" Foreground="#6C7086" FontSize="12"
                       Text="Power Mode isn't available on this machine." Visibility="Collapsed"/>
        </StackPanel>

        <Button Grid.Row="2" Content="Close" HorizontalAlignment="Right" Margin="0,14,0,0"
                Style="{StaticResource ActionButton}" IsCancel="True" Click="Close_Click"/>
    </Grid>
</Window>
```
