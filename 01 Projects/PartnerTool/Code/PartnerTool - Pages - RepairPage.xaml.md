---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\RepairPage.xaml
---

# PartnerTool\Pages\RepairPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.RepairPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:pt="clr-namespace:PartnerTool">

    <UserControl.Resources>
        <!-- Inline log scrollers forward the mouse wheel to the page when they can't scroll further -->
        <Style TargetType="ScrollViewer">
            <Setter Property="pt:ScrollChaining.Enabled" Value="True"/>
        </Style>
        <Style x:Key="MaintBtn" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="MinWidth" Value="165"/>
            <Setter Property="Margin" Value="0,0,8,8"/>
        </Style>
        <!-- Per-section log: hidden until its action runs, then shows that fix's output inline. -->
        <Style x:Key="SectionLog" TargetType="Border">
            <Setter Property="Background" Value="#11111B"/>
            <Setter Property="CornerRadius" Value="6"/>
            <Setter Property="Padding" Value="10,8"/>
            <Setter Property="Margin" Value="0,10,0,0"/>
            <Setter Property="Visibility" Value="Collapsed"/>
        </Style>
        <Style x:Key="SectionLogText" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#9399B2"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="FontFamily" Value="Consolas"/>
            <Setter Property="TextWrapping" Value="Wrap"/>
        </Style>
    </UserControl.Resources>

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <StackPanel Margin="20,16,20,16">

            <!-- QUICK FIXES (incl. maintenance housekeeping) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="QUICK FIXES" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="One-click fixes. No reboot unless noted."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,0,0,10"/>
                    <WrapPanel>
                        <Button x:Name="BtnEmptyBin"        Content="Empty Recycle Bin"     Style="{StaticResource MaintBtn}" Click="EmptyBin_Click"/>
                        <Button x:Name="BtnRestartSpooler"  Content="Restart Print Spooler" Style="{StaticResource MaintBtn}" Click="RestartSpooler_Click"/>
                        <Button x:Name="BtnClearQueue"      Content="Clear Print Queue"     Style="{StaticResource MaintBtn}" Click="ClearQueue_Click"/>
                        <Button x:Name="BtnRestartExplorer" Content="Restart Explorer"      Style="{StaticResource MaintBtn}" Click="RestartExplorer_Click"/>
                        <Button x:Name="BtnRestartAudio"    Content="Restart Audio"         Style="{StaticResource MaintBtn}" Click="RestartAudio_Click"/>
                        <Button x:Name="BtnReregStore"      Content="Re-register Store"      Style="{StaticResource MaintBtn}" Click="ReregStore_Click"/>
                        <Button x:Name="BtnClearIconCache"  Content="Clear Icon Cache"       Style="{StaticResource MaintBtn}" Click="ClearIconCache_Click"/>
                        <Button x:Name="BtnMemDiag"         Content="Memory Diagnostic"      Style="{StaticResource MaintBtn}" Click="MemDiag_Click"/>
                        <!-- Content is set from the current state in RefreshDevMode() -->
                        <Button x:Name="BtnDevMode"         Content="Developer Mode"         Style="{StaticResource MaintBtn}" Click="DevMode_Click"/>
                    </WrapPanel>
                    <TextBlock x:Name="TxtQuickFixStatus" FontSize="11" Margin="0,4,0,0"/>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="QuickFixLogScroll" Height="140" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="QuickFixLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- SYSTEM RESTORE POINTS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" Orientation="Horizontal">
                            <Button x:Name="BtnRestorePoint" Content="Create Restore Point" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="RestorePoint_Click"/>
                            <Button x:Name="BtnOpenRestore" Content="Open System Restore" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="OpenRestore_Click"/>
                            <Button x:Name="BtnRefreshRestore" Content="Refresh" Style="{StaticResource ActionButton}"
                                    Click="RefreshRestore_Click"/>
                        </StackPanel>
                        <TextBlock Text="SYSTEM RESTORE POINTS" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock x:Name="TxtRestoreStatus" FontSize="11" Margin="0,4,0,0"/>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="RestoreLogScroll" Height="100" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="RestoreLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                    <ItemsControl x:Name="IcRestorePoints" Margin="0,6,0,0">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,4">
                                    <TextBlock DockPanel.Dock="Left" Width="150" Foreground="#9399B2" FontSize="11"
                                               VerticalAlignment="Top"
                                               Text="{Binding Created, StringFormat={}{0:MM/dd/yyyy HH:mm}, Mode=OneWay}"/>
                                    <TextBlock DockPanel.Dock="Right" Text="{Binding Type}" Foreground="#6C7086"
                                               FontSize="11" Margin="10,0,0,0" VerticalAlignment="Top"/>
                                    <TextBlock Text="{Binding Description}" Foreground="#CDD6F4" FontSize="12" TextWrapping="Wrap"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoRestore" Foreground="#6C7086" FontSize="12"
                               Text="No restore points found (System Restore may be disabled)." Visibility="Collapsed"/>
                </StackPanel>
            </Border>

            <!-- REPORTS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="REPORTS" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Generate Windows' built-in HTML reports and open them in your browser."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,0,0,10"/>
                    <WrapPanel>
                        <Button x:Name="BtnBatteryReport" Content="Battery Report"      Style="{StaticResource MaintBtn}" Click="BatteryReport_Click"/>
                        <Button x:Name="BtnGpReport"      Content="Group Policy Report" Style="{StaticResource MaintBtn}" Click="GpReport_Click"/>
                    </WrapPanel>
                    <TextBlock x:Name="TxtReportStatus" FontSize="11" Margin="0,4,0,0"/>
                </StackPanel>
            </Border>

            <!-- COLLECT DIAGNOSTICS - a report, not a repair, so it lives with Reports -->
            <Border Style="{StaticResource Card}">
                <DockPanel>
                    <Button x:Name="BtnCollectDiag" DockPanel.Dock="Right" Content="Create Bundle"
                            Style="{StaticResource ActionButton}" VerticalAlignment="Top" Click="CollectDiag_Click"/>
                    <StackPanel Margin="0,0,16,0">
                        <TextBlock Text="COLLECT DIAGNOSTICS" Style="{StaticResource CardTitle}"/>
                        <TextBlock Text="Everything the tool collects in one .zip - HTML report, text summary and the tool's logs."
                                   FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        <TextBlock x:Name="TxtCollectStatus" FontSize="11" Margin="0,4,0,0"/>
                    </StackPanel>
                </DockPanel>
            </Border>

            <!-- ══ Sections below are sorted alphabetically ══ -->

            <!-- ADVANCED CLEANUP (no reboot) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnCleanup" DockPanel.Dock="Right" Content="Run"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                                Click="Cleanup_Click"/>
                        <Button x:Name="BtnCleanupCancel" DockPanel.Dock="Right" Content="Cancel"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top" Margin="0,0,8,0"
                                Visibility="Collapsed" Click="CancelServicing_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="ADVANCED CLEANUP (WinSxS + WMI)" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Shrinks the component store (DISM StartComponentCleanup) and verifies the WMI repository. Frees space, no reboot."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtCleanupStatus" FontSize="11" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="CleanupLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="CleanupLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- CHECK DISK (CHKDSK) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" VerticalAlignment="Top">
                            <ComboBox x:Name="CmbChkdskDrive" Width="150" Visibility="Collapsed"
                                      Style="{StaticResource DarkCombo}" Margin="0,0,0,8"
                                      SelectionChanged="ChkdskDrive_Changed"
                                      ToolTip="Which volume to check"/>
                            <Button x:Name="BtnChkdsk" Content="Scan"
                                    Style="{StaticResource ActionButton}" Click="Chkdsk_Click"/>
                            <Button x:Name="BtnChkdskCancel" Content="Cancel"
                                    Style="{StaticResource ActionButton}" Margin="0,8,0,0"
                                    Visibility="Collapsed" Click="CancelServicing_Click"/>
                            <Button x:Name="BtnSchedChkdsk" Content="Repair /f /r"
                                    Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="SchedChkdsk_Click"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="CHECK DISK (CHKDSK)" Style="{StaticResource CardTitle}"/>
                            <TextBlock x:Name="TxtChkdskHelp"
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtChkdskStatus" FontSize="11" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="ChkdskLogScroll" Height="140" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="ChkdskLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- CLEAN TEMP FILES (ALL USERS) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" VerticalAlignment="Top">
                            <Button x:Name="BtnScanTemp"  Content="Scan"  Style="{StaticResource ActionButton}" Click="ScanTemp_Click"/>
                            <Button x:Name="BtnCleanTemp" Content="Clean" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="CleanTemp_Click"
                                    IsEnabled="False" ToolTip="Run Scan first to see what would be removed"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="CLEAN TEMP FILES (ALL USERS)" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Temp and cache folders for every profile plus the system temp. Scan first. Skips files in use; leaves browser caches, the Office cache and documents alone."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtCleanTempStatus" FontSize="11" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="CleanTempLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="CleanTempLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- DELL SUPPORTASSIST (SYSTEM REPAIR) - only shown on Dell hardware -->
            <Border x:Name="CardDell" Style="{StaticResource Card}" Visibility="Collapsed">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" VerticalAlignment="Top">
                            <Button x:Name="BtnDellRefresh" Content="Scan" Style="{StaticResource ActionButton}" Click="DellRefresh_Click"/>
                            <Button x:Name="BtnDellCapVss"  Content="Cap Shadow Storage" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="DellCapVss_Click"/>
                            <Button x:Name="BtnDellOpenSa"  Content="Open SupportAssist" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="DellOpenSa_Click"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="DELL SUPPORTASSIST (SYSTEM REPAIR)" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Dell OS Recovery snapshots can grow past 100 GB (known purge bug). Don't delete the folder by hand - turn System Repair off in SupportAssist and it purges on the next reboot. Capping VSS shadow storage is Dell's own fix (KB 000129138) - ⚠ it discards existing System Restore points."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtDellStatus" FontSize="11" Margin="0,4,0,0" TextWrapping="Wrap"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="DellLogScroll" Height="120" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="DellLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- FEATURE-UPDATE LEFTOVERS (Disk Cleanup's big categories) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" VerticalAlignment="Top">
                            <Button x:Name="BtnScanFeatUpd"  Content="Scan"  Style="{StaticResource ActionButton}" Click="ScanFeatUpd_Click"/>
                            <Button x:Name="BtnCleanFeatUpd" Content="Clean" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="CleanFeatUpd_Click"
                                    IsEnabled="False" ToolTip="Run Scan first to see what would be removed"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="FEATURE-UPDATE LEFTOVERS" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Windows.old, upgrade staging and the Delivery Optimization cache - often 15-30 GB after a feature update. Scan first. ⚠ Removal is PERMANENT and ends the ~10-day 'go back to the previous Windows' option."
                                       FontSize="11" Foreground="#F9E2AF" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtFeatUpdStatus" FontSize="11" Margin="0,4,0,0" TextWrapping="Wrap"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="FeatUpdLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="FeatUpdLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- OFFICE LANGUAGE PACKS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnRemoveOfficeLangs" DockPanel.Dock="Right" Content="Remove Extra Languages"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top" Click="RemoveOfficeLangs_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="OFFICE LANGUAGE PACKS" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Removes every non-English Office (Click-to-Run) language pack, keeping all en-* variants. Lists what it will remove first. Office apps close during removal; no reboot."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtOfficeLangStatus" FontSize="11" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="OfficeLangLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="OfficeLangLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- PROXY / AUTO-DETECT (WPAD) - Outlook/Office connectivity -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="PROXY / AUTO-DETECT (WPAD)" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Fixes Outlook/Office connectivity after a firewall clears Internet Options ▸ LAN ▸ &quot;Automatically detect settings&quot;. Reset WinHTTP clears a stale system proxy. Signed-in user only."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,8"/>
                    <WrapPanel>
                        <Button x:Name="BtnFixWpad" Content="Re-enable Auto-Detect (WPAD)" Style="{StaticResource MaintBtn}" Click="FixWpad_Click"/>
                        <Button x:Name="BtnResetWinhttp" Content="Reset WinHTTP proxy" Style="{StaticResource MaintBtn}" Click="ResetWinhttp_Click"/>
                    </WrapPanel>
                    <TextBlock x:Name="TxtProxyStatus" FontSize="11" Foreground="#6C7086" Margin="0,2,0,0"/>
                </StackPanel>
            </Border>

            <!-- SYSTEM FILE & IMAGE REPAIR (core, no reboot) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnFullRepair" DockPanel.Dock="Right" Content="Run Full Repair"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                                Click="FullRepair_Click"/>
                        <Button x:Name="BtnFullRepairCancel" DockPanel.Dock="Right" Content="Cancel"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top" Margin="0,0,8,0"
                                Visibility="Collapsed" Click="CancelServicing_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="SYSTEM FILE &amp; IMAGE REPAIR" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="DISM CheckHealth → ScanHealth → RestoreHealth, then SFC /scannow - DISM first, so SFC repairs from a good component store. 15-40 minutes, with live progress. No reboot."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>

                    <StackPanel x:Name="PnlTasks" Visibility="Collapsed" Margin="0,16,0,0">
                        <ItemsControl x:Name="TaskList">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,5">
                                        <TextBlock DockPanel.Dock="Right" Text="{Binding StatusText}"
                                                   Foreground="{Binding StatusColor}" FontSize="11"
                                                   MinWidth="200" TextAlignment="Right" VerticalAlignment="Center"/>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"
                                                   VerticalAlignment="Center"/>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="FullRepairLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="FullRepairLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- WINDOWS INSTALLER CLEANUP (ADOBE BLOAT) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="WINDOWS INSTALLER CLEANUP (ADOBE)" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Reclaims C:\Windows\Installer space when Adobe (or another app) orphans patch files - that hidden folder can reach hundreds of GB. Removes only files no installed product references, and aborts if it can't read that list. Prevent Recurrence sets Adobe's PatchCleanFlag. Scan is read-only."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,8"/>
                    <WrapPanel>
                        <Button x:Name="BtnScanInstaller"  Content="1. Scan (preview)"                 Style="{StaticResource MaintBtn}" Click="ScanInstaller_Click"/>
                        <Button x:Name="BtnCleanInstaller" Content="2. Clean Orphaned Files"           Style="{StaticResource MaintBtn}" Click="CleanInstaller_Click"
                                IsEnabled="False" ToolTip="Run Scan first to see what would be removed"/>
                        <Button x:Name="BtnAdobePatchFix"  Content="3. Prevent Recurrence (Adobe fix)" Style="{StaticResource MaintBtn}" Click="AdobePatchFix_Click"/>
                    </WrapPanel>
                    <TextBlock x:Name="TxtInstallerStatus" FontSize="11" Margin="0,4,0,0" TextWrapping="Wrap"/>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="InstallerLogScroll" Height="170" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="InstallerLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- WINDOWS UPDATE RESET (no reboot) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnWuReset" DockPanel.Dock="Right" Content="Reset"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                                Click="WuReset_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="WINDOWS UPDATE RESET" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Clears the SoftwareDistribution and catroot2 caches and restarts the update services. Fixes stuck Windows Update / Store downloads. No reboot."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <!-- Opt-in: clears the WSUS redirect / update-blocking policies. See the
                                 Updates tab's UPDATE SOURCE card for what is currently set. -->
                            <CheckBox x:Name="ChkWuSource" Margin="0,8,0,0" Foreground="#CDD6F4" FontSize="11"
                                      Content="Also reset the update source back to Windows Update (online)"
                                      ToolTip="Clears the WSUS redirect and the update-blocking policy values."/>
                            <TextBlock Text="Use when UPDATE SOURCE points at a dead WSUS server - the usual cause of &quot;we couldn't connect to the update service&quot;. Intune/MDM is untouched, and a live domain GPO will put it back at the next refresh."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="24,3,0,0"/>
                            <TextBlock x:Name="TxtWuStatus" FontSize="11" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Border Style="{StaticResource SectionLog}">
                        <ScrollViewer x:Name="WuLogScroll" Height="150" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="WuLog" Style="{StaticResource SectionLogText}"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
