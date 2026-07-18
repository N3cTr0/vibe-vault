---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ListWindow.xaml
---

# PartnerTool\ListWindow.xaml

```xml
<Window x:Class="PartnerTool.ListWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Details"
        Width="660" Height="580"
        WindowStartupLocation="CenterOwner"
        Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <TextBlock x:Name="TxtTitle" Grid.Row="0" FontSize="16" FontWeight="SemiBold" Foreground="#CDD6F4"/>

        <ListBox Grid.Row="1" x:Name="Rows" Style="{StaticResource PlainList}" Margin="0,12,0,0"
                 Background="Transparent" BorderThickness="0">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <DockPanel Margin="0,5">
                        <TextBlock DockPanel.Dock="Right" Text="{Binding Right}" Foreground="#9399B2"
                                   FontSize="11" Margin="10,0,0,0" VerticalAlignment="Top" TextAlignment="Right"/>
                        <StackPanel>
                            <TextBlock Text="{Binding Main}" Foreground="#CDD6F4" FontSize="12" TextWrapping="Wrap"/>
                            <TextBlock Text="{Binding Sub}" Foreground="#6C7086" FontSize="11" Margin="0,2,0,0"/>
                        </StackPanel>
                    </DockPanel>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>

        <Button Grid.Row="2" Content="Close" HorizontalAlignment="Right" Margin="0,12,0,0"
                Style="{StaticResource ActionButton}" IsCancel="True" Click="Close_Click"/>
    </Grid>
</Window>
```
