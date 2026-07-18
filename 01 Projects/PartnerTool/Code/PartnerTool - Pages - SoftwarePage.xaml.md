---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SoftwarePage.xaml
---

# PartnerTool\Pages\SoftwarePage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.SoftwarePage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <UserControl.Resources>
        <Style x:Key="CatTitle" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#CBA6F7"/>
            <Setter Property="FontSize" Value="11"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="Margin" Value="0,12,0,4"/>
        </Style>
        <Style x:Key="AppCheck" TargetType="CheckBox">
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="Width" Value="210"/>
            <Setter Property="Margin" Value="0,3"/>
            <Setter Property="FontSize" Value="12"/>
        </Style>
    </UserControl.Resources>

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <StackPanel Margin="20,16,20,16">

            <!-- ══ INSTALL SOFTWARE ══ -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnInstall" DockPanel.Dock="Right" Content="Install Selected"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                                Click="InstallSelected_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="INSTALL SOFTWARE" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Tick the apps to install, then click Install Selected. Everything installs silently via winget. Apps already on this PC are ticked and locked."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                            <TextBlock x:Name="TxtInstallStatus" FontSize="11" Foreground="#89B4FA"
                                       TextWrapping="Wrap" Margin="0,6,0,0"/>
                        </StackPanel>
                    </DockPanel>

                    <Grid Margin="0,8,0,0">
                        <StackPanel x:Name="PnlInstall">

                        <TextBlock Text="BROWSERS" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Google Chrome"   Tag="Google.Chrome"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Microsoft Edge"  Tag="Microsoft.Edge"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Mozilla Firefox" Tag="Mozilla.Firefox"/>
                        </WrapPanel>

                        <TextBlock Text="COMMUNICATIONS" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Slack"           Tag="SlackTechnologies.Slack"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Microsoft Teams" Tag="Microsoft.Teams"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Zoom Workplace"  Tag="Zoom.Zoom"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Nextiva"         Tag="Nextiva.NextivaONE"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="RingCentral"     Tag="RingCentral.RingCentral"/>
                        </WrapPanel>

                        <TextBlock Text="VPN" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="SonicWall NetExtender" Tag="SonicWall.NetExtender"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="OpenVPN Connect"       Tag="OpenVPNTechnologies.OpenVPNConnect"/>
                        </WrapPanel>

                        <TextBlock Text="SECURITY" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Duo Authentication" Tag="DuoSecurity.Duo2FAAuthenticationforWindows"/>
                        </WrapPanel>

                        <TextBlock Text="MICROSOFT TOOLS" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content=".NET Desktop Runtime 6" Tag="Microsoft.DotNet.DesktopRuntime.6"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content=".NET Desktop Runtime 7" Tag="Microsoft.DotNet.DesktopRuntime.7"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content=".NET Desktop Runtime 8" Tag="Microsoft.DotNet.DesktopRuntime.8"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content=".NET Desktop Runtime 9" Tag="Microsoft.DotNet.DesktopRuntime.9"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content=".NET Desktop Runtime 10" Tag="Microsoft.DotNet.DesktopRuntime.10"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="PowerToys"        Tag="Microsoft.PowerToys"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Process Explorer" Tag="Microsoft.Sysinternals.ProcessExplorer"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Process Monitor"  Tag="Microsoft.Sysinternals.ProcessMonitor"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="TCPView"          Tag="Microsoft.Sysinternals.TCPView"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Windows App"      Tag="Microsoft.WindowsApp"/>
                        </WrapPanel>

                        <TextBlock Text="MULTIMEDIA" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Adobe Acrobat Reader" Tag="Adobe.Acrobat.Reader.64-bit"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Foxit Reader"         Tag="Foxit.FoxitReader"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="NAPS2 (Scanner)"      Tag="Cyanfish.NAPS2"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="VLC"                  Tag="VideoLAN.VLC"/>
                        </WrapPanel>

                        <TextBlock Text="PRO TOOLS" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Advanced IP Scanner"        Tag="Famatech.AdvancedIPScanner"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Angry IP Scanner"          Tag="angryziber.AngryIPScanner"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="CPU-Z"                     Tag="CPUID.CPU-Z"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="GPU-Z"                     Tag="TechPowerUp.GPU-Z"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Display Driver Uninstaller" Tag="Wagnardsoft.DisplayDriverUninstaller"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="HWiNFO"                    Tag="REALiX.HWiNFO"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="HWMonitor"                 Tag="CPUID.HWMonitor"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Nmap"                      Tag="Insecure.Nmap"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="PuTTY"                     Tag="PuTTY.PuTTY"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="WinSCP"                    Tag="WinSCP.WinSCP"/>
                        </WrapPanel>

                        <TextBlock Text="UTILITIES" Style="{StaticResource CatTitle}"/>
                        <WrapPanel>
                            <CheckBox Style="{StaticResource AppCheck}" Content="7-Zip"             Tag="7zip.7zip"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="AnyDesk"           Tag="AnyDesk.AnyDesk"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="CrystalDiskInfo"   Tag="CrystalDewWorld.CrystalDiskInfo"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="CrystalDiskMark"   Tag="CrystalDewWorld.CrystalDiskMark"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Google Drive"      Tag="Google.GoogleDrive"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="Revo Uninstaller"  Tag="RevoUninstaller.RevoUninstaller"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="TreeSize Free"     Tag="JAMSoftware.TreeSize.Free"/>
                            <CheckBox Style="{StaticResource AppCheck}" Content="WizTree"           Tag="AntibodySoftware.WizTree"/>
                        </WrapPanel>
                        </StackPanel>

                        <!-- Loading overlay — covers the list while we detect installed apps (first visit only) -->
                        <Border x:Name="InstallLoadingOverlay" Background="#313244" Visibility="Collapsed">
                            <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center" Margin="0,40">
                                <TextBlock Text="Checking for installed software…" Foreground="#CDD6F4"
                                           FontSize="13" FontWeight="SemiBold" HorizontalAlignment="Center"/>
                                <TextBlock Text="This can take up to ~30 seconds the first time."
                                           Foreground="#6C7086" FontSize="11" Margin="0,6,0,0" HorizontalAlignment="Center"/>
                                <ProgressBar IsIndeterminate="True" Width="220" Height="4" Margin="0,16,0,0"
                                             Background="#45475A" Foreground="#89B4FA" BorderThickness="0"/>
                            </StackPanel>
                        </Border>
                    </Grid>

                    <StackPanel x:Name="PnlLog" Visibility="Collapsed" Margin="0,12,0,0">
                        <DockPanel Margin="0,0,0,6">
                            <CheckBox x:Name="ChkAutoScroll" DockPanel.Dock="Right"
                                      Content="Auto-scroll" IsChecked="True"
                                      Foreground="#6C7086" FontSize="10" VerticalAlignment="Center"/>
                            <TextBlock Text="INSTALL LOG" Style="{StaticResource CardTitle}" VerticalAlignment="Center" Margin="0"/>
                        </DockPanel>
                        <Border Background="#11111B" CornerRadius="6" Padding="10,8">
                            <ScrollViewer x:Name="LogScroll" Height="140" VerticalScrollBarVisibility="Auto">
                                <TextBlock x:Name="TxtLog" Foreground="#6C7086" FontSize="10"
                                           FontFamily="Consolas" TextWrapping="Wrap"/>
                            </ScrollViewer>
                        </Border>
                    </StackPanel>
                </StackPanel>
            </Border>

            <!-- ══ STARTUP PROGRAMS (merged from the old Inventory tab) ══ -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnStartupRefresh" DockPanel.Dock="Right" Content="Refresh"
                                Style="{StaticResource ActionButton}" Click="StartupRefresh_Click"/>
                        <TextBlock Text="STARTUP PROGRAMS" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock Text="Programs that launch at sign-in. Disable the ones a machine doesn't need to speed up boot."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,0,0,8"/>
                    <ItemsControl x:Name="IcStartup">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,4">
                                    <Button DockPanel.Dock="Right" Content="{Binding ToggleText}" Tag="{Binding}"
                                            Style="{StaticResource ActionButton}" Padding="12,4" FontSize="11"
                                            Click="ToggleStartup_Click"/>
                                    <Ellipse DockPanel.Dock="Left" Width="9" Height="9" VerticalAlignment="Center" Margin="0,0,10,0">
                                        <Ellipse.Style>
                                            <Style TargetType="Ellipse">
                                                <Setter Property="Fill" Value="#A6E3A1"/>
                                                <Style.Triggers>
                                                    <DataTrigger Binding="{Binding Enabled}" Value="False">
                                                        <Setter Property="Fill" Value="#6C7086"/>
                                                    </DataTrigger>
                                                </Style.Triggers>
                                            </Style>
                                        </Ellipse.Style>
                                    </Ellipse>
                                    <StackPanel>
                                        <TextBlock FontSize="12">
                                            <Run Text="{Binding Name, Mode=OneWay}" Foreground="#CDD6F4"/>
                                            <Run Text="  ·  " Foreground="#6C7086"/>
                                            <Run Text="{Binding StateText, Mode=OneWay}" Foreground="#6C7086"/>
                                            <Run Text="  ·  " Foreground="#6C7086"/>
                                            <Run Text="{Binding LocationText, Mode=OneWay}" Foreground="#6C7086"/>
                                        </TextBlock>
                                        <TextBlock Text="{Binding Command}" Foreground="#6C7086" FontSize="11"
                                                   Margin="0,1,0,0" TextTrimming="CharacterEllipsis"/>
                                    </StackPanel>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoStartup" Foreground="#6C7086" FontSize="12"
                               Text="No startup programs found." Visibility="Collapsed"/>
                </StackPanel>
            </Border>

            <!-- ══ INSTALLED SOFTWARE ══ -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel Margin="0,0,0,8">
                        <TextBlock x:Name="TxtCount" DockPanel.Dock="Right"
                                   Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                        <TextBlock Text="INSTALLED SOFTWARE" Style="{StaticResource CardTitle}"
                                   VerticalAlignment="Center" Margin="0"/>
                    </DockPanel>

                    <!-- Search bar -->
                    <Border Background="#11111B" CornerRadius="6" Padding="12,0" Margin="0,0,0,8">
                        <!-- Placeholder note: the Background must be set via the Style (NOT as a local
                             attribute) — a local value beats a trigger Setter, so an inline
                             Background="Transparent" would stop the placeholder brush from ever showing. -->
                        <TextBox x:Name="SearchBox" Foreground="#CDD6F4"
                                 CaretBrush="#CDD6F4" FontSize="12" BorderThickness="0"
                                 VerticalContentAlignment="Center" Height="36"
                                 Text="" TextChanged="Search_Changed">
                            <TextBox.Style>
                                <Style TargetType="TextBox">
                                    <Style.Resources>
                                        <VisualBrush x:Key="ph" Stretch="None" AlignmentX="Left">
                                            <VisualBrush.Visual>
                                                <TextBlock Text="Type to search installed software…" Foreground="#6C7086" FontSize="12"/>
                                            </VisualBrush.Visual>
                                        </VisualBrush>
                                    </Style.Resources>
                                    <Setter Property="Background" Value="Transparent"/>
                                    <Style.Triggers>
                                        <!-- Show the placeholder only while empty AND not focused, so it
                                             clears the moment the field is clicked into. -->
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

                    <!-- Column headers -->
                    <Grid Margin="0,0,0,2">
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="100"/>
                            <ColumnDefinition Width="160"/>
                            <ColumnDefinition Width="70"/>
                            <ColumnDefinition Width="90"/>
                        </Grid.ColumnDefinitions>
                        <TextBlock Grid.Column="0" Text="Name"      Style="{StaticResource RowLabel}" Padding="6,0"/>
                        <TextBlock Grid.Column="1" Text="Version"   Style="{StaticResource RowLabel}"/>
                        <TextBlock Grid.Column="2" Text="Publisher" Style="{StaticResource RowLabel}"/>
                        <TextBlock Grid.Column="3" Text="Installed" Style="{StaticResource RowLabel}"/>
                    </Grid>

                    <ListBox x:Name="SoftwareList" Height="360"
                             Background="#1E1E2E" BorderThickness="0"
                             VirtualizingPanel.IsVirtualizing="True"
                             VirtualizingPanel.VirtualizationMode="Recycling"
                             ScrollViewer.HorizontalScrollBarVisibility="Disabled">
                        <ListBox.ItemContainerStyle>
                            <Style TargetType="ListBoxItem">
                                <Setter Property="Padding" Value="0"/>
                                <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
                                <Setter Property="Template">
                                    <Setter.Value>
                                        <ControlTemplate TargetType="ListBoxItem">
                                            <Border x:Name="bg" Background="Transparent" Padding="0,1">
                                                <ContentPresenter/>
                                            </Border>
                                            <ControlTemplate.Triggers>
                                                <Trigger Property="IsMouseOver" Value="True">
                                                    <Setter TargetName="bg" Property="Background" Value="#313244"/>
                                                </Trigger>
                                                <Trigger Property="IsSelected" Value="True">
                                                    <Setter TargetName="bg" Property="Background" Value="#45475A"/>
                                                </Trigger>
                                            </ControlTemplate.Triggers>
                                        </ControlTemplate>
                                    </Setter.Value>
                                </Setter>
                            </Style>
                        </ListBox.ItemContainerStyle>
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <Grid Margin="0,3,12,3">
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition Width="*"/>
                                        <ColumnDefinition Width="100"/>
                                        <ColumnDefinition Width="160"/>
                                        <ColumnDefinition Width="70"/>
                                        <ColumnDefinition Width="90"/>
                                    </Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="{Binding Name}"        Foreground="#CDD6F4" FontSize="12" Padding="6,0" VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Version}"     Foreground="#6C7086" FontSize="12" VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                    <TextBlock Grid.Column="2" Text="{Binding Publisher}"   Foreground="#6C7086" FontSize="12" VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                    <TextBlock Grid.Column="3" Text="{Binding InstallDate}" Foreground="#6C7086" FontSize="12" VerticalAlignment="Center"/>
                                    <Button Grid.Column="4" Content="Uninstall" Tag="{Binding}" IsEnabled="{Binding CanUninstall}"
                                            Click="UninstallApp_Click" Style="{StaticResource ActionButton}"
                                            Padding="0,3" Margin="6,0,0,0" FontSize="11"/>
                                </Grid>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
