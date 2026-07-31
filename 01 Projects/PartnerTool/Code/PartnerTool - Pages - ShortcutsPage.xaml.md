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
                        <Button Content="Network Connections"    Style="{StaticResource ToolButton}" Tag="ncpa.cpl"               Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="ncpa.cpl - enable/disable a NIC or set a static IP."/>
                        <Button Content="Remote Desktop"          Style="{StaticResource ToolButton}" Tag="mstsc.exe"              Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="mstsc - connect out to another machine."/>
                        <Button Content="Remote Desktop Settings" Style="{StaticResource ToolButton}" Tag="ms-settings:remotedesktop" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Turn RDP on or off and see who may connect."/>
                        <Button Content="Windows Firewall"        Style="{StaticResource ToolButton}" Tag="wf.msc"                 Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Inbound/outbound rules and per-profile firewall settings."/>
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
                                Tag="explorer://shell:::{A8A91A66-3A7D-4424-8D24-04E180695C7A}" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Classic printers view - set the default, open a queue, troubleshoot."/>
                        <Button x:Name="BtnPrintMgmt" Content="Print Management" Style="{StaticResource ToolButton}"
                                Click="PrintMgmt_Click" Margin="0,0,8,8"
                                ToolTip="Printers, drivers and ports. Optional feature - offered if missing."/>
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
                        <Button Content="Add/Remove Programs"   Style="{StaticResource ToolButton}" Tag="appwiz.cpl"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Programs and Features - uninstall desktop apps."/>
                        <Button Content="Control Panel"         Style="{StaticResource ToolButton}" Tag="control.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Classic Control Panel, for what Settings doesn't cover."/>
                        <Button Content="Credential Manager"    Style="{StaticResource ToolButton}" Tag="explorer://shell:::{1206F5F1-0569-412C-8FEC-3204630DFB70}" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Clear a stale cached password failing against a share or Office."/>
                        <Button Content="Environment Variables" Style="{StaticResource ToolButton}" Tag="rundll32://sysdm.cpl,EditEnvironmentVariables" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="User and system variables, including PATH."/>
                        <Button Content="Group Policy"          Style="{StaticResource ToolButton}" Tag="gpedit.msc"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Local policy only, not domain GPOs. Not on Home editions."/>
                        <Button Content="Registry Editor"       Style="{StaticResource ToolButton}" Tag="regedit.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Edits apply immediately and can't be undone - export first."/>
                        <Button Content="System Configuration"  Style="{StaticResource ToolButton}" Tag="msconfig.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="msconfig - boot options and safe boot, for startup problems."/>
                        <Button Content="System Properties"     Style="{StaticResource ToolButton}" Tag="sysdm.cpl"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Computer name, domain join and System Restore settings."/>
                    </UniformGrid>
                </StackPanel>
            </Border>

            <!-- SYSTEM -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="SYSTEM" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <Button Content="Computer Management" Style="{StaticResource ToolButton}" Tag="compmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Event Viewer, Shares, Users, Devices and Disks in one tree."/>
                        <Button Content="Device Manager (admin)" Style="{StaticResource ToolButton}" Tag="devmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Opens elevated, so devices can be uninstalled - a normal one is read-only."/>
                        <Button Content="DirectX Diagnostic"  Style="{StaticResource ToolButton}" Tag="dxdiag.exe"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="dxdiag - graphics, sound and input detail; saves a report."/>
                        <Button Content="Disk Management"     Style="{StaticResource ToolButton}" Tag="diskmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Assign a missing drive letter or bring a disk online."/>
                        <Button Content="Event Viewer"        Style="{StaticResource ToolButton}" Tag="eventvwr.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The full Windows logs, when Diagnostics isn't enough."/>
                        <Button Content="Performance Monitor" Style="{StaticResource ToolButton}" Tag="perfmon.exe"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="perfmon - long-run counter logging, for intermittent problems."/>
                        <Button Content="Resource Monitor"    Style="{StaticResource ToolButton}" Tag="resmon.exe"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Per-process CPU, disk, network and memory, plus what each is touching."/>
                        <Button Content="Services"            Style="{StaticResource ToolButton}" Tag="services.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="services.msc - including the ones this tool blocks."/>
                        <Button Content="Task Manager"        Style="{StaticResource ToolButton}" Tag="taskmgr.exe"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Windows Task Manager. The Performance window covers the same ground."/>
                    </UniformGrid>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
