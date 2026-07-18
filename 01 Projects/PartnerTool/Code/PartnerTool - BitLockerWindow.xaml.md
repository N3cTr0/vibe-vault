---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\BitLockerWindow.xaml
---

# PartnerTool\BitLockerWindow.xaml

```xml
<Window x:Class="PartnerTool.BitLockerWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="BitLocker Recovery Key"
        Width="560" SizeToContent="Height" MaxHeight="640"
        ResizeMode="NoResize" WindowStartupLocation="CenterOwner"
        Background="#1E1E2E" Icon="Resources/logo.ico">

    <Grid Margin="22">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <StackPanel Grid.Row="0">
            <TextBlock Text="BitLocker Recovery Key" FontSize="16" FontWeight="SemiBold" Foreground="#CDD6F4"/>
            <TextBlock Text="⚠ The 48-digit recovery password below unlocks the drive. Treat it as sensitive — only store it somewhere secure."
                       Foreground="#F9E2AF" FontSize="11" Margin="0,6,0,0" TextWrapping="Wrap"/>
        </StackPanel>

        <ScrollViewer Grid.Row="1" Margin="0,14,0,0" MaxHeight="420" VerticalScrollBarVisibility="Auto">
            <StackPanel>
                <ItemsControl x:Name="KeyList">
                    <ItemsControl.ItemTemplate>
                        <DataTemplate>
                            <Border Background="#313244" CornerRadius="8" Padding="14,12" Margin="0,0,0,10">
                                <StackPanel>
                                    <DockPanel>
                                        <TextBlock DockPanel.Dock="Left" Foreground="#CBA6F7" FontWeight="Bold"
                                                   FontSize="12" Text="{Binding Drive, StringFormat='Drive {0}'}"/>
                                        <TextBlock Foreground="#6C7086" FontSize="10" HorizontalAlignment="Right"
                                                   TextTrimming="CharacterEllipsis"
                                                   Text="{Binding Identifier, StringFormat='ID {0}'}"/>
                                    </DockPanel>
                                    <DockPanel Margin="0,8,0,0">
                                        <Button DockPanel.Dock="Right" Content="Copy" Margin="8,0,0,0"
                                                Style="{StaticResource ActionButton}"
                                                Tag="{Binding RecoveryPassword}" Click="Copy_Click"/>
                                        <Border Background="#11111B" CornerRadius="6" Padding="10,8">
                                            <TextBox Text="{Binding RecoveryPassword}" IsReadOnly="True"
                                                     BorderThickness="0" Background="Transparent"
                                                     Foreground="#A6E3A1" FontFamily="Consolas" FontSize="13"
                                                     TextWrapping="Wrap"/>
                                        </Border>
                                    </DockPanel>
                                </StackPanel>
                            </Border>
                        </DataTemplate>
                    </ItemsControl.ItemTemplate>
                </ItemsControl>

                <TextBlock x:Name="TxtStatus" Foreground="#6C7086" FontSize="12" TextWrapping="Wrap"
                           Text="Reading recovery keys…"/>
            </StackPanel>
        </ScrollViewer>

        <DockPanel Grid.Row="2" Margin="0,14,0,0">
            <TextBlock x:Name="TxtCopied" DockPanel.Dock="Left" Foreground="#A6E3A1" FontSize="11"
                       VerticalAlignment="Center"/>
            <Button DockPanel.Dock="Right" Content="Close" HorizontalAlignment="Right"
                    Style="{StaticResource ActionButton}" Click="Close_Click"/>
        </DockPanel>
    </Grid>
</Window>
```
