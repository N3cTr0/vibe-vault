---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TechGateWindow.xaml
---

# PartnerTool\TechGateWindow.xaml

```xml
<Window x:Class="PartnerTool.TechGateWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Tech Verification" Width="400" SizeToContent="Height"
        Background="#1E1E2E" WindowStartupLocation="CenterOwner"
        ResizeMode="NoResize" ShowInTaskbar="False">
    <StackPanel Margin="22,20">
        <TextBlock Text="TECH VERIFICATION" Foreground="#CBA6F7" FontSize="12" FontWeight="Bold" Margin="0,0,0,8"/>
        <TextBlock x:Name="TxtPrompt" Foreground="#CDD6F4" FontSize="12" TextWrapping="Wrap" Margin="0,0,0,14">
            This action can change or remove data on this machine.<LineBreak/>Enter tech code to continue.
        </TextBlock>
        <Border Background="#45475A" CornerRadius="6" Padding="10,0">
            <PasswordBox x:Name="TxtCode" Height="34" Background="Transparent" Foreground="#CDD6F4" FontSize="14"
                         BorderThickness="0" VerticalContentAlignment="Center" MaxLength="12"/>
        </Border>
        <TextBlock x:Name="TxtErr" Foreground="#F38BA8" FontSize="11" Margin="2,6,0,0" Visibility="Collapsed"/>
        <DockPanel Margin="0,18,0,0">
            <Button DockPanel.Dock="Right" Content="Proceed" IsDefault="True" Click="Ok_Click"
                    Style="{StaticResource ActionButton}" Padding="18,7" Margin="8,0,0,0"/>
            <Button DockPanel.Dock="Right" Content="Cancel" IsCancel="True" Click="Cancel_Click"
                    Style="{StaticResource ActionButton}" Padding="18,7"/>
            <TextBlock/>
        </DockPanel>
    </StackPanel>
</Window>
```
