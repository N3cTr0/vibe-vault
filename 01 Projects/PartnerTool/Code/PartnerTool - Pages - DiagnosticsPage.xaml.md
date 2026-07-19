---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\DiagnosticsPage.xaml
---

# PartnerTool\Pages\DiagnosticsPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.DiagnosticsPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <UserControl.Resources>
        <!-- Event severity dot -->
        <Style x:Key="EventDot" TargetType="Ellipse">
            <Setter Property="Width" Value="8"/>
            <Setter Property="Height" Value="8"/>
            <Setter Property="VerticalAlignment" Value="Top"/>
            <Setter Property="Margin" Value="0,4,10,0"/>
            <Setter Property="Fill" Value="#F9E2AF"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding Level}" Value="Critical">
                    <Setter Property="Fill" Value="#F38BA8"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </UserControl.Resources>

    <Grid Background="#1E1E2E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Auto-refresh status (no manual button — refreshes when the page is opened) -->
        <DockPanel Grid.Row="0" Margin="20,12,20,4">
            <TextBlock x:Name="TxtRefreshed" Foreground="#6C7086" FontSize="11"
                       VerticalAlignment="Center"/>
        </DockPanel>

        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
            <StackPanel Margin="20,4,20,16">

                <!-- RELIABILITY -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <DockPanel>
                            <Button x:Name="BtnLoadReliability" DockPanel.Dock="Right" Content="Load detailed history"
                                    Style="{StaticResource ActionButton}" Click="LoadReliability_Click"/>
                            <TextBlock Text="RELIABILITY (SYSTEM STABILITY)" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                        </DockPanel>
                        <TextBlock x:Name="TxtStability" FontSize="26" FontWeight="Bold"
                                   Foreground="#A6E3A1" Margin="0,6,0,2"/>
                        <TextBlock x:Name="TxtStabilityNote" Foreground="#6C7086" FontSize="11"
                                   TextWrapping="Wrap" Margin="0,0,0,2"/>
                    </StackPanel>
                </Border>

                <!-- CRASH HISTORY — directly under Reliability, because its caption points here -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="CRASH HISTORY" Style="{StaticResource CardTitle}"/>

                        <!-- Recent crash dump = a .dmp file currently on disk (history below is the event log) -->
                        <DockPanel Margin="0,0,0,8">
                            <TextBlock DockPanel.Dock="Left" Text="Recent crash dump:" Foreground="#6C7086" FontSize="11"
                                       VerticalAlignment="Top" Margin="0,0,8,0"/>
                            <TextBlock x:Name="TxtDumps" FontSize="11" Foreground="#CDD6F4" TextWrapping="Wrap"/>
                        </DockPanel>
                        <Rectangle Style="{StaticResource RowDivider}" Margin="0,0,0,8"/>

                        <TextBlock Text="BLUE SCREENS (BUGCHECKS)" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                        <ItemsControl x:Name="IcBsod">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,4">
                                        <TextBlock DockPanel.Dock="Left" Width="120" VerticalAlignment="Top" Foreground="#9399B2" FontSize="11"
                                                   Text="{Binding Time, StringFormat='{}{0:MM/dd/yyyy HH:mm}', Mode=OneWay}"/>
                                        <StackPanel>
                                            <TextBlock Text="{Binding StopCode}" Foreground="#F38BA8" FontSize="11" FontWeight="SemiBold"/>
                                            <TextBlock Text="{Binding Detail}" Foreground="#6C7086" FontSize="11" TextWrapping="Wrap"/>
                                        </StackPanel>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                        <TextBlock x:Name="TxtNoBsod" Text="● No blue-screen bugchecks recorded" Foreground="#A6E3A1" FontSize="12" Visibility="Collapsed"/>

                        <Rectangle Style="{StaticResource RowDivider}" Margin="0,8"/>
                        <TextBlock Text="APPLICATION CRASHES" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                        <ItemsControl x:Name="IcAppCrash">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,3">
                                        <TextBlock DockPanel.Dock="Left" Width="120" Foreground="#9399B2" FontSize="11"
                                                   Text="{Binding Time, StringFormat='{}{0:MM/dd/yyyy HH:mm}', Mode=OneWay}"/>
                                        <TextBlock Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis">
                                            <Run Text="{Binding App, Mode=OneWay}"/>
                                            <Run Text="  ·  " Foreground="#6C7086"/><Run Text="{Binding Module, Mode=OneWay}" Foreground="#6C7086"/><Run Text="{Binding Note, Mode=OneWay}" Foreground="#F9E2AF"/>
                                        </TextBlock>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                        <TextBlock x:Name="TxtNoAppCrash" Text="● No application crashes recorded" Foreground="#A6E3A1" FontSize="12" Visibility="Collapsed"/>
                    </StackPanel>
                </Border>

                <!-- HEALTH HISTORY (trend across launches) -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <DockPanel>
                            <Button x:Name="BtnMoreHistory" DockPanel.Dock="Right" Content="More…"
                                    Style="{StaticResource ActionButton}" Padding="12,4" FontSize="11"
                                    Click="MoreHistory_Click" Visibility="Collapsed"/>
                            <TextBlock Text="HEALTH HISTORY (LAST 7 DAYS)" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                        </DockPanel>
                        <Grid Margin="0,2,0,2">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="120"/>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="*"/>
                            </Grid.ColumnDefinitions>
                            <TextBlock Grid.Column="0" Text="Date"      Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="1" Text="Disk free" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="2" Text="RAM"       Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="3" Text="Batt wear" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="4" Text="Stability" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        </Grid>
                        <ItemsControl x:Name="IcHistory">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <Grid Margin="0,3">
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="120"/>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="*"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Foreground="#9399B2" FontSize="11"
                                                   Text="{Binding When, StringFormat={}{0:MM/dd/yyyy}, Mode=OneWay}"/>
                                        <TextBlock Grid.Column="1" Foreground="#CDD6F4" FontSize="11"
                                                   Text="{Binding DiskFreeGb, StringFormat={}{0:F0} GB, Mode=OneWay}"/>
                                        <TextBlock Grid.Column="2" Foreground="#CDD6F4" FontSize="11"
                                                   Text="{Binding RamPct, StringFormat={}{0:F0}%, Mode=OneWay}"/>
                                        <TextBlock Grid.Column="3" Foreground="#CDD6F4" FontSize="11"
                                                   Text="{Binding BatteryWearPct, StringFormat={}{0}%, Mode=OneWay, TargetNullValue=—}"/>
                                        <TextBlock Grid.Column="4" Foreground="#CDD6F4" FontSize="11"
                                                   Text="{Binding StabilityIndex, StringFormat={}{0:F1}, Mode=OneWay, TargetNullValue=—}"/>
                                    </Grid>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                        <TextBlock x:Name="TxtNoHistory" Text="History starts building from this launch."
                                   Foreground="#6C7086" FontSize="12" Visibility="Collapsed"/>
                    </StackPanel>
                </Border>

                <!-- DEVICE PROBLEMS -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="DEVICE PROBLEMS" Style="{StaticResource CardTitle}"/>
                        <TextBlock x:Name="TxtNoDevices" Text="● No devices reporting problems"
                                   Foreground="#A6E3A1" FontSize="12" Visibility="Collapsed"/>
                        <ItemsControl x:Name="IcDevices">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,4">
                                        <Ellipse DockPanel.Dock="Left" Width="8" Height="8" Fill="#F9E2AF"
                                                 VerticalAlignment="Center" Margin="0,0,10,0"/>
                                        <TextBlock DockPanel.Dock="Right" Text="{Binding Problem}"
                                                   Foreground="#F9E2AF" FontSize="11" VerticalAlignment="Center"
                                                   Margin="12,0,0,0"/>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"
                                                   VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </Border>

                <!-- TOP PROCESSES (live) -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <DockPanel>
                            <Button x:Name="BtnProcRefresh" DockPanel.Dock="Right" Content="Refresh"
                                    Style="{StaticResource ActionButton}" Click="ProcRefresh_Click"/>
                            <TextBlock Text="TOP PROCESSES" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                        </DockPanel>
                        <Grid Margin="0,4,0,2">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="70"/>
                                <ColumnDefinition Width="70"/>
                                <ColumnDefinition Width="90"/>
                                <ColumnDefinition Width="60"/>
                            </Grid.ColumnDefinitions>
                            <TextBlock Grid.Column="0" Text="Process"  Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="1" Text="PID"      Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                            <TextBlock Grid.Column="2" Text="CPU"      Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                            <TextBlock Grid.Column="3" Text="Memory"   Foreground="#6C7086" FontSize="10" FontWeight="Bold" TextAlignment="Right"/>
                            <TextBlock Grid.Column="4" Text=""         />
                        </Grid>
                        <ItemsControl x:Name="IcProcesses">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <Grid Margin="0,3">
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="70"/>
                                            <ColumnDefinition Width="70"/>
                                            <ColumnDefinition Width="90"/>
                                            <ColumnDefinition Width="60"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"
                                                   VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                        <TextBlock Grid.Column="1" Text="{Binding Pid}" Foreground="#6C7086"
                                                   FontSize="11" TextAlignment="Right" VerticalAlignment="Center"/>
                                        <TextBlock Grid.Column="2" Text="{Binding CpuPct, StringFormat={}{0:F0}%}" Foreground="#9399B2"
                                                   FontSize="12" TextAlignment="Right" VerticalAlignment="Center"/>
                                        <TextBlock Grid.Column="3" Text="{Binding MemMb, StringFormat={}{0:F0} MB}" Foreground="#9399B2"
                                                   FontSize="12" TextAlignment="Right" VerticalAlignment="Center"/>
                                        <Button Grid.Column="4" Content="End" Tag="{Binding Pid}" Click="KillProc_Click"
                                                IsEnabled="{Binding CanEnd}"
                                                Style="{StaticResource ActionButton}" Padding="0,4" Margin="8,0,0,0" FontSize="11"/>
                                    </Grid>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </Border>

                <!-- POWER & RESTART HISTORY -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="POWER &amp; RESTART HISTORY (LAST 14 DAYS)" Style="{StaticResource CardTitle}"/>
                        <ItemsControl x:Name="IcPowerEvents">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,4">
                                        <TextBlock DockPanel.Dock="Left" Width="120" VerticalAlignment="Top"
                                                   Foreground="#9399B2" FontSize="11"
                                                   Text="{Binding Time, StringFormat={}{0:MM/dd/yyyy HH:mm}, Mode=OneWay}"/>
                                        <StackPanel>
                                            <TextBlock Text="{Binding Kind}" Foreground="#CDD6F4" FontSize="11" FontWeight="SemiBold"/>
                                            <TextBlock Text="{Binding Detail}" Foreground="#6C7086" FontSize="11"
                                                       TextWrapping="Wrap" Margin="0,1,0,0"/>
                                        </StackPanel>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                        <TextBlock x:Name="TxtNoPower" Text="No power events recorded."
                                   Foreground="#6C7086" FontSize="12" Visibility="Collapsed"/>
                    </StackPanel>
                </Border>

                <!-- BOOT PERFORMANCE -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="BOOT PERFORMANCE" Style="{StaticResource CardTitle}"/>
                        <TextBlock x:Name="TxtBootLatest" Foreground="#CDD6F4" FontSize="13" Margin="0,0,0,6"/>
                        <TextBlock Text="TOP SLOW-BOOT CONTRIBUTORS" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,2,0,4"/>
                        <ItemsControl x:Name="IcBootContrib">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,3">
                                        <TextBlock DockPanel.Dock="Right" Foreground="#F9E2AF" FontSize="11"
                                                   Text="{Binding Seconds, StringFormat='{}{0:F1} s', Mode=OneWay}"/>
                                        <TextBlock DockPanel.Dock="Left" Width="60" Foreground="#6C7086" FontSize="11" Text="{Binding Kind}"/>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                        <TextBlock x:Name="TxtNoBoot" Foreground="#6C7086" FontSize="12"
                                   Text="No boot-performance data (the Diagnostics-Performance log may be disabled)." Visibility="Collapsed"/>
                    </StackPanel>
                </Border>

                <!-- RECENT ERRORS -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <DockPanel>
                            <Button x:Name="BtnMoreEvents" DockPanel.Dock="Right" Content="More…"
                                    Style="{StaticResource ActionButton}" Padding="12,4" FontSize="11"
                                    Click="MoreEvents_Click" Visibility="Collapsed"/>
                            <TextBlock Text="RECENT ERRORS — CRITICAL &amp; ERROR (SYSTEM &amp; APPLICATION)"
                                       Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                        </DockPanel>
                        <TextBlock x:Name="TxtNoEvents" Text="● No critical or error events in the last 10 days"
                                   Foreground="#A6E3A1" FontSize="12" Visibility="Collapsed"/>
                        <ItemsControl x:Name="IcEvents">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,6">
                                        <Ellipse Style="{StaticResource EventDot}" DockPanel.Dock="Left"/>
                                        <StackPanel>
                                            <TextBlock FontSize="11">
                                                <Run Text="{Binding Time, StringFormat={}{0:MM/dd/yyyy HH:mm}, Mode=OneWay}" Foreground="#9399B2"/>
                                                <Run Text="  ·  "/>
                                                <Run Text="{Binding Source, Mode=OneWay}" Foreground="#CDD6F4"/>
                                                <Run Text="{Binding Id, StringFormat=(ID {0}), Mode=OneWay}" Foreground="#6C7086"/>
                                            </TextBlock>
                                            <TextBlock Text="{Binding Message}" Foreground="#6C7086"
                                                       FontSize="11" TextWrapping="Wrap" Margin="0,2,0,0"/>
                                        </StackPanel>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </Border>

            </StackPanel>
        </ScrollViewer>
    </Grid>
</UserControl>
```
