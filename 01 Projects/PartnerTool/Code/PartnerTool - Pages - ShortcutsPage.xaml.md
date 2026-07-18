---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ShortcutsPage.xaml
---

# PartnerTool\Pages\ShortcutsPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.ShortcutsPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <!-- Categories are ordered alphabetically; the buttons inside each are too. -->
        <StackPanel Margin="20,16,20,16">

            <!-- NETWORK & REMOTE -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="NETWORK &amp; REMOTE" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <Button Content="Network Connections"    Style="{StaticResource ToolButton}" Tag="ncpa.cpl"               Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Remote Desktop"          Style="{StaticResource ToolButton}" Tag="mstsc.exe"              Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Remote Desktop Settings" Style="{StaticResource ToolButton}" Tag="ms-settings:remotedesktop" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Windows Firewall"        Style="{StaticResource ToolButton}" Tag="wf.msc"                 Click="Tool_Click" Margin="0,0,8,8"/>
                    </UniformGrid>
                </StackPanel>
            </Border>

            <!-- PRINTERS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="PRINTERS" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <!-- Classic Control Panel "Devices and Printers" (not the Settings page) -->
                        <Button Content="Devices &amp; Printers" Style="{StaticResource ToolButton}"
                                Tag="explorer://shell:::{A8A91A66-3A7D-4424-8D24-04E180695C7A}" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button x:Name="BtnPrintMgmt" Content="Print Management" Style="{StaticResource ToolButton}"
                                Click="PrintMgmt_Click" Margin="0,0,8,8"/>
                    </UniformGrid>
                    <ProgressBar x:Name="PrintProgress" Height="6" Margin="0,10,0,0" Minimum="0" Maximum="100"
                                 Visibility="Collapsed" Background="#313244" Foreground="#A6E3A1" BorderThickness="0"/>
                    <TextBlock x:Name="TxtPrintStatus" FontSize="11" Margin="0,6,0,0"
                               Visibility="Collapsed" TextWrapping="Wrap"/>
                </StackPanel>
            </Border>

            <!-- SETTINGS & ADMIN -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="SETTINGS &amp; ADMIN" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <Button Content="Add/Remove Programs"   Style="{StaticResource ToolButton}" Tag="appwiz.cpl"  Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Control Panel"         Style="{StaticResource ToolButton}" Tag="control.exe" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Environment Variables" Style="{StaticResource ToolButton}" Tag="rundll32://sysdm.cpl,EditEnvironmentVariables" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Group Policy"          Style="{StaticResource ToolButton}" Tag="gpedit.msc"  Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Registry Editor"       Style="{StaticResource ToolButton}" Tag="regedit.exe" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="System Properties"     Style="{StaticResource ToolButton}" Tag="sysdm.cpl"   Click="Tool_Click" Margin="0,0,8,8"/>
                    </UniformGrid>
                </StackPanel>
            </Border>

            <!-- SYSTEM -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="SYSTEM" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <Button Content="Computer Management" Style="{StaticResource ToolButton}" Tag="compmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Device Manager"      Style="{StaticResource ToolButton}" Tag="devmgmt.msc"  Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Disk Management"     Style="{StaticResource ToolButton}" Tag="diskmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Event Viewer"        Style="{StaticResource ToolButton}" Tag="eventvwr.msc" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Services"            Style="{StaticResource ToolButton}" Tag="services.msc" Click="Tool_Click" Margin="0,0,8,8"/>
                        <Button Content="Task Manager"        Style="{StaticResource ToolButton}" Tag="taskmgr.exe"  Click="Tool_Click" Margin="0,0,8,8"/>
                    </UniformGrid>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
