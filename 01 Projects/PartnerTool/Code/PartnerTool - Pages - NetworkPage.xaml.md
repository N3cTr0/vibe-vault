---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\NetworkPage.xaml
---

# PartnerTool\Pages\NetworkPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.NetworkPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:pt="clr-namespace:PartnerTool">

    <UserControl.Resources>
        <!-- Output/log scrollers forward the mouse wheel to the page when they can't scroll further -->
        <Style TargetType="ScrollViewer">
            <Setter Property="pt:ScrollChaining.Enabled" Value="True"/>
        </Style>
        <Style x:Key="NetBtn" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="MinWidth" Value="150"/>
            <Setter Property="Margin" Value="0,0,8,8"/>
        </Style>
    </UserControl.Resources>

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <StackPanel Margin="20,16,20,16">

            <!-- NETWORK TOOLS (moved from the old Actions tab) -->
            <Border Style="{StaticResource Card}">
                <StackPanel x:Name="PnlNetTools">
                    <TextBlock Text="NETWORK TOOLS" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Quick network actions — just click a button (no input needed). Results appear below once you run one."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,8"/>
                    <WrapPanel>
                        <Button Content="Flush DNS"          Style="{StaticResource NetBtn}" Click="FlushDns_Click"/>
                        <Button Content="Release / Renew IP" Style="{StaticResource NetBtn}" Click="RenewIp_Click"/>
                        <Button Content="Show Public IP"     Style="{StaticResource NetBtn}" Click="PublicIp_Click"/>
                        <Button Content="Connectivity Test"  Style="{StaticResource NetBtn}" Click="ConnTest_Click"/>
                        <Button Content="Speed Test"         Style="{StaticResource NetBtn}" Click="SpeedTest_Click"/>
                    </WrapPanel>
                    <Border x:Name="NetLogPanel" Background="#11111B" CornerRadius="6" Padding="10,8" Margin="0,10,0,0"
                            Visibility="Collapsed">
                        <ScrollViewer x:Name="NetLogScroll" Height="92" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="TxtNetLog" Foreground="#6C7086" FontSize="10"
                                       FontFamily="Consolas" TextWrapping="Wrap"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

            <!-- PING & DIAGNOSTIC TOOLS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="PING &amp; DIAGNOSTIC TOOLS" Style="{StaticResource CardTitle}"/>
                    <TextBlock Text="Enter a target, then pick a tool. DNS type applies to DNS Lookup; Port applies to Port Check (one or several — e.g. 80,443,3389 or 1000-1010). Output appears below."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,2,0,10"/>

                    <!-- Only the inputs + tool buttons disable while a tool runs. The output panel
                         must stay OUT of this scope: IsEnabled inherits, and the Stop button lives
                         down there — disabling the whole card disabled the very button meant to
                         interrupt the run (caught by the scripted UI test). -->
                    <StackPanel x:Name="PnlNetDiag">
                    <!-- One aligned input row: Host/IP grows; DNS type + Port are compact -->
                    <Grid Margin="0,0,0,10">
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="Auto"/>
                        </Grid.ColumnDefinitions>
                        <TextBlock Grid.Column="0" Text="Host / IP" Foreground="#9399B2" FontSize="12"
                                   VerticalAlignment="Center" Margin="0,0,10,0"/>
                        <Border Grid.Column="1" Background="#45475A" CornerRadius="6" Padding="10,0">
                            <TextBox x:Name="TxtNetHost" Text="8.8.8.8" Height="32" Background="Transparent"
                                     Foreground="#CDD6F4" FontSize="12" BorderThickness="0" VerticalContentAlignment="Center"/>
                        </Border>
                        <TextBlock Grid.Column="2" Text="DNS type" Foreground="#9399B2" FontSize="12"
                                   VerticalAlignment="Center" Margin="16,0,10,0"/>
                        <Border Grid.Column="3" Background="#45475A" CornerRadius="6" Padding="10,0">
                            <TextBox x:Name="TxtDnsType" Text="A" Width="44" Height="32" Background="Transparent"
                                     Foreground="#CDD6F4" FontSize="12" BorderThickness="0" VerticalContentAlignment="Center"
                                     TextAlignment="Center"/>
                        </Border>
                        <TextBlock Grid.Column="4" Text="Port" Foreground="#9399B2" FontSize="12"
                                   VerticalAlignment="Center" Margin="16,0,10,0"/>
                        <Border Grid.Column="5" Background="#45475A" CornerRadius="6" Padding="10,0">
                            <TextBox x:Name="TxtPort" Text="443" Width="130" Height="32" Background="Transparent"
                                     Foreground="#CDD6F4" FontSize="12" BorderThickness="0" VerticalContentAlignment="Center"
                                     TextAlignment="Center" ToolTip="One or more ports: 443 · 80,443,3389 · 1000-1010"/>
                        </Border>
                    </Grid>

                    <!-- Uniform tool buttons -->
                    <UniformGrid Columns="4">
                        <Button Content="Ping"       AutomationProperties.AutomationId="NetPing"       Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="Ping_Click"/>
                        <Button Content="Traceroute" AutomationProperties.AutomationId="NetTraceroute" Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="Traceroute_Click"/>
                        <Button Content="DNS Lookup" AutomationProperties.AutomationId="NetDnsLookup"  Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="DnsLookup_Click"/>
                        <Button Content="Port Check" AutomationProperties.AutomationId="NetPortCheck"  Style="{StaticResource ActionButton}" Click="PortCheck_Click"/>
                    </UniformGrid>
                    </StackPanel>

                    <Border x:Name="ToolOutPanel" Background="#11111B" CornerRadius="6" Padding="10,8"
                            Margin="0,10,0,0" Visibility="Collapsed">
                        <DockPanel>
                            <DockPanel DockPanel.Dock="Top" Margin="0,0,0,4" LastChildFill="False">
                                <Button x:Name="BtnToolCopy" AutomationProperties.AutomationId="NetToolCopy"
                                        DockPanel.Dock="Right" Content="Copy" Style="{StaticResource ActionButton}"
                                        Padding="10,2" FontSize="11" MinHeight="0" Click="ToolCopy_Click"
                                        ToolTip="Copy the tool output to the clipboard"/>
                                <Button x:Name="BtnToolStop" AutomationProperties.AutomationId="NetToolStop"
                                        DockPanel.Dock="Right" Content="Stop" Style="{StaticResource ActionButton}"
                                        Padding="10,2" FontSize="11" MinHeight="0" Margin="0,0,8,0"
                                        Visibility="Collapsed" Click="ToolStop_Click"
                                        ToolTip="Stop the running trace"/>
                            </DockPanel>
                            <ScrollViewer x:Name="ToolOutScroll" Height="150" VerticalScrollBarVisibility="Auto">
                                <TextBlock x:Name="TxtToolOut" Foreground="#6C7086"
                                           FontSize="11" FontFamily="Consolas" TextWrapping="Wrap"/>
                            </ScrollViewer>
                        </DockPanel>
                    </Border>
                </StackPanel>
            </Border>

            <!-- IP SCANNER -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" Orientation="Horizontal">
                            <Button x:Name="BtnScanStop" Content="Stop" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="ScanStop_Click" Visibility="Collapsed"/>
                            <Button x:Name="BtnScanCopy" Content="Copy" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="ScanCopy_Click" IsEnabled="False"/>
                            <Button x:Name="BtnScan" Content="Scan" Style="{StaticResource ActionButton}" Click="Scan_Click"/>
                        </StackPanel>
                        <TextBlock Text="IP SCANNER" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock Text="Finds live devices on a network — like Advanced IP Scanner. Enter a range as CIDR (192.168.1.0/24), a span (192.168.1.1-254) or a single address. Each host is pinged, then ARP-probed (which finds devices that ignore ping, and gets the MAC), then reverse-DNS'd for a name. Deep scan also knocks on common TCP ports (445/3389/139/80/443/22) and lists each host's open ports — slower, but the only way to spot a ping-blocking device on another subnet."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,10"/>

                    <Grid Margin="0,0,0,10">
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="Auto"/>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="Auto"/>
                        </Grid.ColumnDefinitions>
                        <TextBlock Grid.Column="0" Text="IP range" Foreground="#9399B2" FontSize="12"
                                   VerticalAlignment="Center" Margin="0,0,10,0"/>
                        <Border Grid.Column="1" Background="#45475A" CornerRadius="6" Padding="10,0">
                            <TextBox x:Name="TxtScanRange" Height="32" Background="Transparent"
                                     Foreground="#CDD6F4" FontSize="12" BorderThickness="0"
                                     VerticalContentAlignment="Center" KeyDown="ScanRange_KeyDown"/>
                        </Border>
                        <CheckBox Grid.Column="2" x:Name="ChkScanDeep" Content="Deep scan (TCP ports)"
                                  Foreground="#9399B2" FontSize="11" VerticalAlignment="Center" Margin="16,0,0,0"/>
                    </Grid>

                    <TextBlock x:Name="TxtScanStatus" Foreground="#6C7086" FontSize="11"/>
                    <ProgressBar x:Name="ScanBusy" Height="3" Minimum="0" Maximum="100" Margin="0,4,0,0"
                                 Visibility="Collapsed" Background="#313244" Foreground="#89B4FA" BorderThickness="0"/>

                    <!-- Results. Grid.IsSharedSizeScope + the "Ports" SharedSizeGroup keeps the header
                         and every row's last column the same width; it collapses to nothing on a
                         non-deep scan (header hidden in code + empty cells), so it's not wasted space. -->
                    <StackPanel x:Name="PnlScanResults" Visibility="Collapsed" Margin="0,8,0,0" Grid.IsSharedSizeScope="True">
                        <Grid Margin="0,0,0,2">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="120"/>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="140"/>
                                <ColumnDefinition Width="86"/>
                                <ColumnDefinition Width="Auto" SharedSizeGroup="Ports"/>
                            </Grid.ColumnDefinitions>
                            <TextBlock Grid.Column="0" Text="IP Address" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="1" Text="Name"       Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="2" Text="MAC"        Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="3" Text="Found by"   Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                            <TextBlock Grid.Column="4" x:Name="TxtPortsHeader" Text="Open ports" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        </Grid>
                        <ItemsControl x:Name="IcScanHosts">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <Grid Margin="0,2">
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="120"/>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="140"/>
                                            <ColumnDefinition Width="86"/>
                                            <ColumnDefinition Width="Auto" SharedSizeGroup="Ports"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Foreground="#CDD6F4" FontSize="11" FontFamily="Consolas">
                                            <Run Text="{Binding Ip, Mode=OneWay}"/>
                                        </TextBlock>
                                        <TextBlock Grid.Column="1" FontSize="11" TextTrimming="CharacterEllipsis" Foreground="#CDD6F4">
                                            <Run Text="{Binding NameText, Mode=OneWay}"/>
                                            <Run Text="{Binding NoteText, Mode=OneWay}" Foreground="#89B4FA"/>
                                        </TextBlock>
                                        <TextBlock Grid.Column="2" Text="{Binding MacText}" Foreground="#9399B2" FontSize="11" FontFamily="Consolas"/>
                                        <TextBlock Grid.Column="3" Text="{Binding How}"     Foreground="#6C7086" FontSize="11"/>
                                        <TextBlock Grid.Column="4" Text="{Binding PortsText}" Foreground="#A6E3A1" FontSize="11" FontFamily="Consolas" TextTrimming="CharacterEllipsis"
                                                   Visibility="{Binding Deep, Converter={StaticResource BoolToVis}}"/>
                                    </Grid>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </StackPanel>
            </Border>

            <!-- WI-FI -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <StackPanel DockPanel.Dock="Right" Orientation="Horizontal">
                            <Button x:Name="BtnWifiScan" Content="Scan Nearby" Style="{StaticResource ActionButton}"
                                    Margin="0,0,8,0" Click="WifiScan_Click"/>
                            <Button x:Name="BtnWifiRefresh" Content="Refresh"
                                    Style="{StaticResource ActionButton}" Click="WifiRefresh_Click"/>
                        </StackPanel>
                        <TextBlock Text="WI-FI" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock x:Name="TxtWifiStatus" Foreground="#CDD6F4" FontSize="12" Margin="0,2,0,2" TextWrapping="Wrap"/>
                    <Rectangle Style="{StaticResource RowDivider}" Margin="0,6"/>
                    <TextBlock Text="SAVED NETWORKS" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                    <ItemsControl x:Name="IcWifiProfiles">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,3">
                                    <Button DockPanel.Dock="Right" Content="Show password" Tag="{Binding}"
                                            Style="{StaticResource ActionButton}" Padding="10,4" FontSize="11"
                                            Click="ShowWifiPassword_Click"/>
                                    <TextBlock Text="{Binding}" Foreground="#CDD6F4" FontSize="12"
                                               VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoWifi" Text="No saved Wi-Fi networks (or Wi-Fi not present)."
                               Foreground="#6C7086" FontSize="12" Visibility="Collapsed"/>

                    <!-- Nearby networks (Scan Nearby) — hidden until a scan runs -->
                    <StackPanel x:Name="PnlWifiScan" Visibility="Collapsed">
                        <Rectangle Style="{StaticResource RowDivider}" Margin="0,8,0,6"/>
                        <TextBlock Text="NEARBY NETWORKS" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                        <TextBlock x:Name="TxtWifiScanStatus" Foreground="#6C7086" FontSize="12"/>
                        <ItemsControl x:Name="IcWifiScan" Margin="0,4,0,0">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,3">
                                        <TextBlock DockPanel.Dock="Right" Foreground="#9399B2" FontSize="11">
                                            <Run Text="{Binding Signal, Mode=OneWay}"/><Run Text="  ch "/><Run Text="{Binding Channel, Mode=OneWay}"/>
                                        </TextBlock>
                                        <TextBlock Foreground="#CDD6F4" FontSize="12" TextTrimming="CharacterEllipsis">
                                            <Run Text="{Binding Ssid, Mode=OneWay}"/>
                                            <Run Text="  ·  " Foreground="#6C7086"/><Run Text="{Binding Radio, Mode=OneWay}" Foreground="#6C7086"/>
                                        </TextBlock>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </StackPanel>
            </Border>

            <!-- ACTIVE CONNECTIONS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnConnRefresh" DockPanel.Dock="Right" Content="Refresh"
                                Style="{StaticResource ActionButton}" Click="ConnRefresh_Click"/>
                        <TextBlock x:Name="TxtConnCount" DockPanel.Dock="Right" Foreground="#6C7086"
                                   FontSize="11" VerticalAlignment="Center" Margin="0,0,10,0"/>
                        <TextBlock Text="ACTIVE CONNECTIONS &amp; LISTENING PORTS" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <Border Background="#45475A" CornerRadius="6" Padding="10,0" Margin="0,6,0,4">
                        <TextBox x:Name="TxtConnFilter" Foreground="#CDD6F4" FontSize="12"
                                 BorderThickness="0" Height="30" VerticalContentAlignment="Center"
                                 TextChanged="ConnFilter_TextChanged">
                            <TextBox.Style>
                                <!-- Placeholder pattern (same as the Software page search box). -->
                                <Style TargetType="TextBox">
                                    <Style.Resources>
                                        <VisualBrush x:Key="ph" Stretch="None" AlignmentX="Left">
                                            <VisualBrush.Visual>
                                                <TextBlock Text="Filter by process, address, port, or state (e.g. chrome, 443, LISTENING)…"
                                                           Foreground="#6C7086" FontSize="12"/>
                                            </VisualBrush.Visual>
                                        </VisualBrush>
                                    </Style.Resources>
                                    <Setter Property="Background" Value="Transparent"/>
                                    <Style.Triggers>
                                        <MultiTrigger>
                                            <MultiTrigger.Conditions>
                                                <Condition Property="Text" Value=""/>
                                                <Condition Property="IsKeyboardFocused" Value="False"/>
                                            </MultiTrigger.Conditions>
                                            <Setter Property="Background" Value="{StaticResource ph}"/>
                                        </MultiTrigger>
                                    </Style.Triggers>
                                </Style>
                            </TextBox.Style>
                        </TextBox>
                    </Border>
                    <Grid Margin="0,4,0,2">
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="44"/>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="110"/>
                            <ColumnDefinition Width="*"/>
                        </Grid.ColumnDefinitions>
                        <TextBlock Grid.Column="0" Text="Proto"   Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        <TextBlock Grid.Column="1" Text="Local"   Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        <TextBlock Grid.Column="2" Text="Remote"  Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        <TextBlock Grid.Column="3" Text="State"   Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                        <TextBlock Grid.Column="4" Text="Process" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                    </Grid>
                    <ItemsControl x:Name="IcConnections">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <Grid Margin="0,2">
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition Width="44"/>
                                        <ColumnDefinition Width="*"/>
                                        <ColumnDefinition Width="*"/>
                                        <ColumnDefinition Width="110"/>
                                        <ColumnDefinition Width="*"/>
                                    </Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="{Binding Proto}"   Foreground="#9399B2" FontSize="11"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Local}"   Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                    <TextBlock Grid.Column="2" Text="{Binding Remote}"  Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                    <TextBlock Grid.Column="3" Text="{Binding State}"   Foreground="#6C7086" FontSize="11"/>
                                    <TextBlock Grid.Column="4" Text="{Binding Process}" Foreground="#9399B2" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                </Grid>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                </StackPanel>
            </Border>

            <!-- Adapter cards injected from code-behind (2 columns) -->
            <ItemsControl x:Name="AdapterList">
                <ItemsControl.ItemsPanel>
                    <ItemsPanelTemplate>
                        <UniformGrid Columns="2"/>
                    </ItemsPanelTemplate>
                </ItemsControl.ItemsPanel>
                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <Border Style="{StaticResource Card}" Margin="0,0,12,12">
                            <StackPanel>
                                <TextBlock Text="{Binding Header}" Style="{StaticResource CardTitle}"/>

                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="Type"        Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Type}" Style="{StaticResource RowValue}"/>
                                </Grid>
                                <Rectangle Style="{StaticResource RowDivider}"/>
                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="IP Address"  Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding IpAddress}" Style="{StaticResource RowValue}"/>
                                </Grid>
                                <Rectangle Style="{StaticResource RowDivider}"/>
                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="Subnet Mask" Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding SubnetMask}" Style="{StaticResource RowValue}"/>
                                </Grid>
                                <Rectangle Style="{StaticResource RowDivider}"/>
                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="Gateway"     Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Gateway}" Style="{StaticResource RowValue}"/>
                                </Grid>
                                <Rectangle Style="{StaticResource RowDivider}"/>
                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="DNS Servers" Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Dns}" Style="{StaticResource RowValue}"/>
                                </Grid>
                                <Rectangle Style="{StaticResource RowDivider}"/>
                                <Grid><Grid.ColumnDefinitions><ColumnDefinition Width="140"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="MAC Address" Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Mac}" Style="{StaticResource RowValue}"/>
                                </Grid>
                            </StackPanel>
                        </Border>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>

            <!-- ══ Destructive resets last — risk increases as you scroll ══ -->

            <!-- NETWORK STACK RESET (moved from Repair) -->
            <Border Style="{StaticResource Card}">
                <DockPanel>
                    <Button x:Name="BtnNetReset" DockPanel.Dock="Right" Content="Reset"
                            Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                            Click="NetReset_Click"/>
                    <StackPanel Margin="0,0,16,0">
                        <TextBlock Text="NETWORK STACK RESET" Style="{StaticResource CardTitle}"/>
                        <TextBlock Text="Resets Winsock and the TCP/IP stack and flushes the DNS cache. Fixes broken connectivity without touching your adapters. ✔ You will NOT lose connectivity — your adapters, internet, and remote (Ninja) session stay active. A restart is needed to fully finish, but you can do that later at a convenient time."
                                   FontSize="11" Foreground="#F9E2AF" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        <TextBlock x:Name="TxtNetResetStatus" FontSize="11" Margin="0,4,0,0"/>
                    </StackPanel>
                </DockPanel>
            </Border>

            <!-- NETWORK RESET (Windows-style: reinstalls adapters on reboot) -->
            <Border Style="{StaticResource Card}">
                <DockPanel>
                    <Button x:Name="BtnNetFullReset" DockPanel.Dock="Right" Content="Reset"
                            Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                            Click="NetFullReset_Click"/>
                    <StackPanel Margin="0,0,16,0">
                        <TextBlock Text="NETWORK RESET (REINSTALL ADAPTERS)" Style="{StaticResource CardTitle}"/>
                        <TextBlock Text="The full Windows Network reset: removes every network adapter and resets all networking components to defaults — Windows reinstalls the adapters on the next reboot. ⚠ This WILL drop your remote (Ninja) session. The tool restarts the PC automatically to finish — it reconnects once Windows is back up. Last resort; this clears VPN clients, static IPs and saved Wi-Fi networks."
                                   FontSize="11" Foreground="#F38BA8" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        <TextBlock x:Name="TxtNetFullResetStatus" FontSize="11" Margin="0,4,0,0"/>
                    </StackPanel>
                </DockPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
