---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SmartWindow.xaml
---

# PartnerTool\SmartWindow.xaml

```xml
<Window x:Class="PartnerTool.SmartWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="SMART Details"
        Width="720" Height="620"
        WindowStartupLocation="CenterOwner"
        Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <StackPanel Grid.Row="0">
            <TextBlock Text="SMART DETAILS" FontSize="16" FontWeight="SemiBold" Foreground="#CDD6F4"/>
            <TextBlock Text="Full self-monitoring attribute table for SATA/ATA drives (CrystalDiskInfo-style). NVMe drives don't expose this table — their temperature, wear and power-on hours are on the System Info page."
                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
        </StackPanel>

        <ScrollViewer Grid.Row="1" Margin="0,12,0,0" VerticalScrollBarVisibility="Auto">
            <StackPanel>
                <!-- One card per drive, injected from code-behind -->
                <ItemsControl x:Name="IcDrives">
                    <ItemsControl.ItemTemplate>
                        <DataTemplate>
                            <Border Background="#181825" BorderBrush="#313244" BorderThickness="1" CornerRadius="6"
                                    Padding="12" Margin="0,0,0,12">
                                <StackPanel>
                                    <DockPanel Margin="0,0,0,8">
                                        <TextBlock DockPanel.Dock="Right" Text="{Binding StatusText}" FontSize="11"
                                                   FontWeight="SemiBold" Foreground="{Binding StatusBrush}" VerticalAlignment="Center"/>
                                        <TextBlock Text="{Binding Title}" Foreground="#CDD6F4" FontSize="13"
                                                   FontWeight="SemiBold" TextTrimming="CharacterEllipsis"/>
                                    </DockPanel>

                                    <!-- Column header -->
                                    <Grid Margin="0,0,0,4">
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="52"/>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="64"/>
                                            <ColumnDefinition Width="56"/>
                                            <ColumnDefinition Width="72"/>
                                            <ColumnDefinition Width="120"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Text="ID"        Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                                        <TextBlock Grid.Column="1" Text="Attribute" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                                        <TextBlock Grid.Column="2" Text="Current"   Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                                        <TextBlock Grid.Column="3" Text="Worst"     Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                                        <TextBlock Grid.Column="4" Text="Thresh"    Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                                        <TextBlock Grid.Column="5" Text="Raw"       Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                                    </Grid>

                                    <ItemsControl ItemsSource="{Binding Attributes}">
                                        <ItemsControl.ItemTemplate>
                                            <DataTemplate>
                                                <Grid Margin="0,2">
                                                    <Grid.ColumnDefinitions>
                                                        <ColumnDefinition Width="52"/>
                                                        <ColumnDefinition Width="*"/>
                                                        <ColumnDefinition Width="64"/>
                                                        <ColumnDefinition Width="56"/>
                                                        <ColumnDefinition Width="72"/>
                                                        <ColumnDefinition Width="120"/>
                                                    </Grid.ColumnDefinitions>
                                                    <TextBlock Grid.Column="0" Text="{Binding IdText}"   Foreground="#6C7086" FontSize="11" FontFamily="Consolas"/>
                                                    <TextBlock Grid.Column="1" Text="{Binding Name}"     Foreground="{Binding RowBrush}" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                                    <TextBlock Grid.Column="2" Text="{Binding Current}"  Foreground="{Binding RowBrush}" FontSize="11" TextAlignment="Right"/>
                                                    <TextBlock Grid.Column="3" Text="{Binding Worst}"    Foreground="#9399B2" FontSize="11" TextAlignment="Right"/>
                                                    <TextBlock Grid.Column="4" Text="{Binding ThreshText}" Foreground="#9399B2" FontSize="11" TextAlignment="Right"/>
                                                    <TextBlock Grid.Column="5" Text="{Binding RawText}"  Foreground="#9399B2" FontSize="11" FontFamily="Consolas" TextAlignment="Right"/>
                                                </Grid>
                                            </DataTemplate>
                                        </ItemsControl.ItemTemplate>
                                    </ItemsControl>
                                </StackPanel>
                            </Border>
                        </DataTemplate>
                    </ItemsControl.ItemTemplate>
                </ItemsControl>

                <TextBlock x:Name="TxtEmpty" Foreground="#9399B2" FontSize="12" TextWrapping="Wrap" Visibility="Collapsed"
                           Text="No SATA/ATA SMART tables are available on this machine. NVMe drives don't expose the classic attribute table — their temperature, wear and power-on hours are shown on the System Info page under Storage."/>
            </StackPanel>
        </ScrollViewer>

        <Button Grid.Row="2" Content="Close" HorizontalAlignment="Right" Margin="0,12,0,0"
                Style="{StaticResource ActionButton}" IsCancel="True" Click="Close_Click"/>
    </Grid>
</Window>
```
