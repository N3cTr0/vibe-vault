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
                                ToolTip="Adapter list (ncpa.cpl) - enable/disable a NIC, set a static IP, or check adapter properties."/>
                        <Button Content="Remote Desktop"          Style="{StaticResource ToolButton}" Tag="mstsc.exe"              Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The Remote Desktop client (mstsc) - connect out to another machine."/>
                        <Button Content="Remote Desktop Settings" Style="{StaticResource ToolButton}" Tag="ms-settings:remotedesktop" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Turn Remote Desktop on or off for this PC and see who is allowed to connect in."/>
                        <Button Content="Windows Firewall"        Style="{StaticResource ToolButton}" Tag="wf.msc"                 Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Firewall with Advanced Security - inbound/outbound rules, per-profile settings and connection logging."/>
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
                                ToolTip="The classic Control Panel view, not the Settings page - set the default printer, open a queue, or run printer troubleshooting."/>
                        <Button x:Name="BtnPrintMgmt" Content="Print Management" Style="{StaticResource ToolButton}"
                                Click="PrintMgmt_Click" Margin="0,0,8,8"
                                ToolTip="Manage printers, drivers and ports across the machine. An optional Windows feature - the tool offers to install it if missing."/>
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
                                ToolTip="The classic Programs and Features list - uninstall desktop apps and view installed updates."/>
                        <Button Content="Control Panel"         Style="{StaticResource ToolButton}" Tag="control.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The full classic Control Panel, for the applets Settings still doesn't cover."/>
                        <Button Content="Credential Manager"    Style="{StaticResource ToolButton}" Tag="explorer://shell:::{1206F5F1-0569-412C-8FEC-3204630DFB70}" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Saved Windows and web credentials - clear a stale cached password that keeps failing against a share or Office."/>
                        <Button Content="Environment Variables" Style="{StaticResource ToolButton}" Tag="rundll32://sysdm.cpl,EditEnvironmentVariables" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Windows' own editor for user and system variables, including PATH."/>
                        <Button Content="Group Policy"          Style="{StaticResource ToolButton}" Tag="gpedit.msc"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Local Group Policy Editor - local policy only, not domain GPOs. Not present on Home editions."/>
                        <Button Content="Registry Editor"       Style="{StaticResource ToolButton}" Tag="regedit.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Opens elevated. Edits here take effect immediately and are not undoable - export the key first."/>
                        <Button Content="System Configuration"  Style="{StaticResource ToolButton}" Tag="msconfig.exe" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="msconfig - boot options, safe boot and services, for narrowing down a startup problem."/>
                        <Button Content="System Properties"     Style="{StaticResource ToolButton}" Tag="sysdm.cpl"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Computer name and domain/workgroup join, plus performance, remote and System Restore settings."/>
                    </UniformGrid>
                </StackPanel>
            </Border>

            <!-- SYSTEM -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="SYSTEM" Style="{StaticResource CardTitle}"/>
                    <UniformGrid Columns="3">
                        <Button Content="Computer Management" Style="{StaticResource ToolButton}" Tag="compmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The combined console - Event Viewer, Shared Folders, Local Users and Groups, Device Manager and Disk Management in one tree."/>
                        <Button Content="Device Manager (admin)" Style="{StaticResource ToolButton}" Tag="devmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Opens elevated (inherits this tool's admin rights), so devices can be uninstalled - e.g. removing network adapters for a reinstall. A plain Device Manager opened by a standard user is read-only."/>
                        <Button Content="DirectX Diagnostic"  Style="{StaticResource ToolButton}" Tag="dxdiag.exe"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="dxdiag - graphics, sound and input device detail, and it saves a text report for escalation."/>
                        <Button Content="Disk Management"     Style="{StaticResource ToolButton}" Tag="diskmgmt.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Partitions, drive letters and volume health - assign a missing letter or bring a disk online."/>
                        <Button Content="Event Viewer"        Style="{StaticResource ToolButton}" Tag="eventvwr.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The full Windows logs, when the Diagnostics page's summary isn't enough."/>
                        <Button Content="Performance Monitor" Style="{StaticResource ToolButton}" Tag="perfmon.exe"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="perfmon - long-run counter logging and data collector sets, for problems that need watching over time."/>
                        <Button Content="Resource Monitor"    Style="{StaticResource ToolButton}" Tag="resmon.exe"   Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Live CPU, disk, network and memory broken down per process - shows which file or address a process is actually touching."/>
                        <Button Content="Services"            Style="{StaticResource ToolButton}" Tag="services.msc" Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="The Windows services console - start, stop and set startup type, including services this tool blocks."/>
                        <Button Content="Task Manager"        Style="{StaticResource ToolButton}" Tag="taskmgr.exe"  Click="Tool_Click" Margin="0,0,8,8"
                                ToolTip="Windows' own Task Manager. The tool's Performance window covers the same ground without leaving the app."/>
                    </UniformGrid>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
