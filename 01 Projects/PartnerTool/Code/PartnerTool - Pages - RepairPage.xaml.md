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
                    <TextBlock Text="One-click fixes and quick housekeeping. Empty the recycle bin, fix the print spooler, or restart Explorer/Audio. For temp files, use Clean Temp Files (All Users) below — it has a Scan preview. No reboot unless noted."
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
                            <Button x:Name="BtnRefreshRestore" Content="Refresh" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="RefreshRestore_Click"/>
                            <Button x:Name="BtnOpenRestore" Content="Open System Restore" Style="{StaticResource ActionButton}"
                                    Click="OpenRestore_Click"/>
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

            <!-- COLLECT DIAGNOSTICS — a report, not a repair, so it lives with Reports -->
            <Border Style="{StaticResource Card}">
                <DockPanel>
                    <Button x:Name="BtnCollectDiag" DockPanel.Dock="Right" Content="Create Bundle"
                            Style="{StaticResource ActionButton}" VerticalAlignment="Top" Click="CollectDiag_Click"/>
                    <StackPanel Margin="0,0,16,0">
                        <TextBlock Text="COLLECT DIAGNOSTICS" Style="{StaticResource CardTitle}"/>
                        <TextBlock Text="Bundles a full system report into a single .zip — every section the tool collects: hardware, OS, performance, power, security &amp; hardening, network adapters &amp; saved Wi-Fi, monitors, printers, accounts, installed software, startup programs, Windows Update history and device problems. Great as an 'as-found' record of an old PC when setting up a new one. Includes an HTML report, a text summary and the tool's logs."
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
                            <TextBlock Text="Analyzes and shrinks the component store (DISM StartComponentCleanup) and verifies / salvages the WMI repository. Frees disk space and repairs the WMI store that powers system queries. No reboot required."
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
                            <Button x:Name="BtnChkdsk" Content="Scan"
                                    Style="{StaticResource ActionButton}" Click="Chkdsk_Click"/>
                            <Button x:Name="BtnChkdskCancel" Content="Cancel"
                                    Style="{StaticResource ActionButton}" Margin="0,8,0,0"
                                    Visibility="Collapsed" Click="CancelServicing_Click"/>
                            <Button x:Name="BtnSchedChkdsk" Content="Schedule /f /r"
                                    Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="SchedChkdsk_Click"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="CHECK DISK (CHKDSK)" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Scan runs an online, read-only check of C: for file-system errors (chkdsk C: /scan) — safe, no reboot. If it reports problems, Schedule /f /r queues a full repair that runs at the next restart (the PC is unusable while it runs)."
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
                            <Button x:Name="BtnCleanTemp" Content="Clean" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="CleanTemp_Click"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="CLEAN TEMP FILES (ALL USERS)" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Scan first to see how much space these folders are using, then Clean to delete them. Covers the temp/cache folders for every user profile plus the system temp: AppData\Local\Temp, CrashDumps, Windows Error Reporting, INetCache, D3DSCache, the RDP client cache, and C:\Windows\Temp. Keeps the folders, skips files in use, and won't follow junctions. Does NOT touch browser caches, the Office document cache, or any documents."
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

            <!-- DELL SUPPORTASSIST (SYSTEM REPAIR) — only shown on Dell hardware -->
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
                            <TextBlock Text="Dell's OS Recovery keeps repair snapshots in C:\ProgramData\Dell\SARemediation — 15 GB reserved by default, but a known purge bug can grow it past 100 GB. Dell's guidance is NOT to delete that folder by hand (it breaks OS Recovery): turn System Repair off in SupportAssist and it purges itself on the next reboot. Separately, Dell KB 000129138: VSS ships with no size limit, so shadow copies can silently eat the drive. Capping shadow storage is Dell's own fix — ⚠ it discards existing System Restore points."
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
                            <Button x:Name="BtnCleanFeatUpd" Content="Clean" Style="{StaticResource ActionButton}" Margin="0,8,0,0" Click="CleanFeatUpd_Click"/>
                        </StackPanel>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="FEATURE-UPDATE LEFTOVERS" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Reclaims the big space Windows Disk Cleanup targets that the temp cleaner doesn't: the previous-Windows rollback image and upgrade staging (Windows.old, $Windows.~BT, $Windows.~WS, $GetCurrent) plus the Delivery Optimization peer cache. Often 15–30 GB after a feature update. Scan first to see what's present. ⚠ Removing Windows.old and the staging folders is PERMANENT and ends the ~10-day 'go back to the previous Windows' option."
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
                            <TextBlock Text="Removes every non-English Microsoft 365 / Office (Click-to-Run) language pack and keeps English (all en-* variants). Scans the installed languages, lists exactly what it will remove for confirmation, then uninstalls each one silently using its own Office uninstaller. Office apps are closed during removal. No reboot required."
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

            <!-- PROXY / AUTO-DETECT (WPAD) — Outlook/Office connectivity -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="PROXY / AUTO-DETECT (WPAD)" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Fixes Office/Outlook connectivity when a firewall (e.g. SonicWall) clears Internet Options ▸ LAN ▸ &quot;Automatically detect settings&quot;. Toggling it off→on re-runs WPAD auto-discovery; resetting the WinHTTP proxy clears a stale system proxy that can also break Outlook/Teams. Applies to the signed-in user."
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
                            <TextBlock Text="Runs the standard no-reboot repair sequence: DISM CheckHealth → ScanHealth → RestoreHealth, then SFC /scannow. DISM runs first on purpose — SFC repairs system files from the component store that DISM restores. This can take 15–40 minutes; each step shows live progress and elapsed time."
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
                    <TextBlock Text="Reclaims space from C:\Windows\Installer when Adobe Acrobat/Reader (or another app) leaves orphaned patch files behind — a known bug that can bloat that hidden folder to tens or hundreds of GB. It asks Windows Installer which cached .msi/.msp files are still needed and removes ONLY files no installed product references (so repair/uninstall of installed apps is unaffected); if that list can't be read it aborts. 'Prevent recurrence' sets Adobe's PatchCleanFlag so its updater cleans up going forward. Scan is a read-only preview; cleaning is tech-gated and every file is logged."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,8"/>
                    <WrapPanel>
                        <Button x:Name="BtnAdobePatchFix"  Content="1. Prevent Recurrence (Adobe fix)" Style="{StaticResource MaintBtn}" Click="AdobePatchFix_Click"/>
                        <Button x:Name="BtnScanInstaller"  Content="2. Scan (preview)"                 Style="{StaticResource MaintBtn}" Click="ScanInstaller_Click"/>
                        <Button x:Name="BtnCleanInstaller" Content="3. Clean Orphaned Files"           Style="{StaticResource MaintBtn}" Click="CleanInstaller_Click"/>
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
                            <TextBlock Text="Stops the update services and clears the SoftwareDistribution and catroot2 caches, then restarts them. Fixes stuck Windows Update / Microsoft Store downloads. No reboot required."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
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
