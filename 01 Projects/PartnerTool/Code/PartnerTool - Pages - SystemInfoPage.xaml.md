---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SystemInfoPage.xaml
---

# PartnerTool\Pages\SystemInfoPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.SystemInfoPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <UserControl.Resources>
        <!-- Live-metric tile (inset well inside the card) -->
        <!-- Clickable live-stat tile. A Button (not a bare Border) so it's keyboard-focusable,
             announced by screen readers via AutomationProperties.Name, and addressable by scripted
             tests — a plain Border never enters the UIA control tree at all. Templated to look exactly
             like the old tile, with a hover/focus highlight. -->
        <Style x:Key="LiveTileButton" TargetType="Button">
            <Setter Property="Background" Value="#1E1E2E"/>
            <Setter Property="Padding" Value="14,10"/>
            <Setter Property="Margin" Value="3"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="bd" Background="{TemplateBinding Background}" CornerRadius="6"
                                Padding="{TemplateBinding Padding}">
                            <ContentPresenter HorizontalAlignment="Stretch"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="bd" Property="Background" Value="#242536"/>
                            </Trigger>
                            <Trigger Property="IsKeyboardFocused" Value="True">
                                <Setter TargetName="bd" Property="Background" Value="#242536"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
        <Style x:Key="LiveLabel" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#9399B2"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
        </Style>
        <Style x:Key="LiveValue" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="FontSize" Value="20"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="Margin" Value="0,2,0,0"/>
        </Style>
        <!-- Disk health dot, coloured by the Health string -->
        <Style x:Key="HealthDot" TargetType="Ellipse">
            <Setter Property="Width" Value="9"/>
            <Setter Property="Height" Value="9"/>
            <Setter Property="VerticalAlignment" Value="Center"/>
            <Setter Property="Margin" Value="0,0,10,0"/>
            <Setter Property="Fill" Value="#A6E3A1"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding Health}" Value="Warning">
                    <Setter Property="Fill" Value="#F9E2AF"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding Health}" Value="Unhealthy">
                    <Setter Property="Fill" Value="#F38BA8"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding Health}" Value="Unknown">
                    <Setter Property="Fill" Value="#6C7086"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </UserControl.Resources>

    <Grid Background="#1E1E2E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Auto-refresh status (no manual button — the page refreshes itself) -->
        <DockPanel Grid.Row="0" Margin="20,12,20,4">
            <TextBlock x:Name="TxtRefreshed" Foreground="#6C7086" FontSize="11"
                       VerticalAlignment="Center"/>
        </DockPanel>

        <!-- 2-column content area -->
        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
            <StackPanel Margin="20,4,20,16">

            <!-- TOP ROW: live performance (left) + power (right) -->
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="12"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <!-- LIVE PERFORMANCE (2 on top, 2 on bottom) -->
                <Border Grid.Column="0" Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="LIVE STATS" Style="{StaticResource CardTitle}"/>
                        <UniformGrid Rows="2" Columns="2">
                            <Button Style="{StaticResource LiveTileButton}" Tag="cpu" Click="LiveTile_Click"
                                    AutomationProperties.AutomationId="LiveTileCpu" AutomationProperties.Name="CPU — open Performance monitor">
                                <StackPanel>
                                    <TextBlock Text="CPU" Style="{StaticResource LiveLabel}"/>
                                    <TextBlock x:Name="TxtLiveCpu" Text="—" Style="{StaticResource LiveValue}" Foreground="#89B4FA"/>
                                    <Border x:Name="LiveCpuPlot" Height="40" Margin="0,6,0,0" Background="#11111B" CornerRadius="4" ClipToBounds="True">
                                        <Polyline x:Name="LiveCpuLine" Stroke="#89B4FA" StrokeThickness="1.5"/>
                                    </Border>
                                </StackPanel>
                            </Button>
                            <Button Style="{StaticResource LiveTileButton}" Tag="mem" Click="LiveTile_Click"
                                    AutomationProperties.AutomationId="LiveTileMem" AutomationProperties.Name="Memory — open Performance monitor">
                                <StackPanel>
                                    <TextBlock Text="MEMORY" Style="{StaticResource LiveLabel}"/>
                                    <TextBlock x:Name="TxtLiveRam" Text="—" Style="{StaticResource LiveValue}" Foreground="#A6E3A1"/>
                                    <Border x:Name="LiveRamPlot" Height="40" Margin="0,6,0,0" Background="#11111B" CornerRadius="4" ClipToBounds="True">
                                        <Polyline x:Name="LiveRamLine" Stroke="#A6E3A1" StrokeThickness="1.5"/>
                                    </Border>
                                </StackPanel>
                            </Button>
                            <Button Style="{StaticResource LiveTileButton}" Tag="disk" Click="LiveTile_Click"
                                    AutomationProperties.AutomationId="LiveTileDisk" AutomationProperties.Name="Disk — open Performance monitor">
                                <StackPanel>
                                    <TextBlock Text="DISK" Style="{StaticResource LiveLabel}"/>
                                    <TextBlock x:Name="TxtLiveDisk" Text="—" Style="{StaticResource LiveValue}" Foreground="#F9E2AF"/>
                                    <Border x:Name="LiveDiskPlot" Height="40" Margin="0,6,0,0" Background="#11111B" CornerRadius="4" ClipToBounds="True">
                                        <Polyline x:Name="LiveDiskLine" Stroke="#F9E2AF" StrokeThickness="1.5"/>
                                    </Border>
                                </StackPanel>
                            </Button>
                            <Button Style="{StaticResource LiveTileButton}" Tag="net" Click="LiveTile_Click"
                                    AutomationProperties.AutomationId="LiveTileNet" AutomationProperties.Name="Network — open Performance monitor">
                                <StackPanel>
                                    <TextBlock Text="NETWORK (Mbps)" Style="{StaticResource LiveLabel}"/>
                                    <TextBlock x:Name="TxtLiveNet" Text="—" Style="{StaticResource LiveValue}" FontSize="15" Foreground="#CBA6F7"/>
                                    <Border x:Name="LiveNetPlot" Height="40" Margin="0,6,0,0" Background="#11111B" CornerRadius="4" ClipToBounds="True">
                                        <Polyline x:Name="LiveNetLine" Stroke="#CBA6F7" StrokeThickness="1.5"/>
                                    </Border>
                                </StackPanel>
                            </Button>
                        </UniformGrid>
                    </StackPanel>
                </Border>

                <!-- POWER -->
                <Border Grid.Column="2" Style="{StaticResource Card}" MinWidth="320">
                    <StackPanel>
                        <TextBlock Text="POWER" Style="{StaticResource CardTitle}"/>
                        <UniformGrid Rows="2" Columns="2">
                            <Button Content="Restart"   Style="{StaticResource ActionButton}" Margin="0,0,5,5" Padding="0,6" FontSize="12" Click="Restart_Click"/>
                            <Button Content="Shut Down" Style="{StaticResource ActionButton}" Margin="5,0,0,5" Padding="0,6" FontSize="12" Click="Shutdown_Click"/>
                            <Button Content="Lock"      Style="{StaticResource ActionButton}" Margin="0,5,5,0" Padding="0,6" FontSize="12" Click="Lock_Click"/>
                            <Button Content="Sign Out"  Style="{StaticResource ActionButton}" Margin="5,5,0,0" Padding="0,6" FontSize="12" Click="SignOut_Click"/>
                        </UniformGrid>

                        <!-- Power status (painted from the startup snapshot) -->
                        <Rectangle Style="{StaticResource RowDivider}" Margin="0,10,0,8"/>
                        <TextBlock Text="POWER STATUS" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,6"/>
                        <Grid>
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="96"/>
                                <ColumnDefinition Width="*"/>
                            </Grid.ColumnDefinitions>
                            <Grid.RowDefinitions>
                                <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                <RowDefinition/><RowDefinition/><RowDefinition/>
                            </Grid.RowDefinitions>
                            <TextBlock Grid.Row="0" Grid.Column="0" Text="Source"        Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="0" Grid.Column="1" x:Name="TxtPwrSource"  Foreground="#CDD6F4" FontSize="11" Margin="0,2" TextTrimming="CharacterEllipsis"/>
                            <TextBlock Grid.Row="1" Grid.Column="0" Text="Power mode"    Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="1" Grid.Column="1" Foreground="#CDD6F4" FontSize="11" Margin="0,2">
                                <Run x:Name="TxtPwrMode"/><Run Text="   "/><Hyperlink Click="ChangePowerPlan_Click" Foreground="#89B4FA">(change)</Hyperlink>
                            </TextBlock>
                            <TextBlock Grid.Row="2" Grid.Column="0" Text="Fast startup"  Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="2" Grid.Column="1" Foreground="#CDD6F4" FontSize="11" Margin="0,2" TextTrimming="CharacterEllipsis">
                                <Run x:Name="TxtPwrFast"/><Run Text="   "/><Hyperlink Click="ChangeStartupSettings_Click" Foreground="#89B4FA" ToolTip="Open 'Choose what the power buttons do' — fast startup + hibernation live here">(change)</Hyperlink>
                            </TextBlock>
                            <TextBlock Grid.Row="3" Grid.Column="0" Text="Hibernation"   Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="3" Grid.Column="1" Foreground="#CDD6F4" FontSize="11" Margin="0,2">
                                <Run x:Name="TxtPwrHib"/><Run Text="   "/><Hyperlink Click="HibToggle_Click" Foreground="#89B4FA" ToolTip="Toggle hibernation directly (powercfg /hibernate). Turning it off also disables fast startup."><Run x:Name="LnkHibToggle" Text="(toggle)"/></Hyperlink>
                            </TextBlock>
                            <TextBlock Grid.Row="4" Grid.Column="0" Text="Sleep after"   Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="4" Grid.Column="1" Foreground="#CDD6F4" FontSize="11" Margin="0,2" TextTrimming="CharacterEllipsis">
                                <Run x:Name="TxtPwrSleep"/><Run Text="   "/><Hyperlink Click="ChangeSleepSettings_Click" Foreground="#89B4FA" ToolTip="Open Windows Settings — Power &amp; sleep">(change)</Hyperlink>
                            </TextBlock>
                            <TextBlock Grid.Row="5" Grid.Column="0" Text="Display off"   Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="5" Grid.Column="1" x:Name="TxtPwrDisplay" Foreground="#CDD6F4" FontSize="11" Margin="0,2" TextTrimming="CharacterEllipsis"/>
                            <TextBlock Grid.Row="6" Grid.Column="0" Text="Last wake"     Foreground="#6C7086" FontSize="11" Margin="0,2"/>
                            <TextBlock Grid.Row="6" Grid.Column="1" x:Name="TxtPwrWake"    Foreground="#CDD6F4" FontSize="11" Margin="0,2"
                                       TextTrimming="CharacterEllipsis" ToolTip="{Binding Text, RelativeSource={RelativeSource Self}}"/>
                        </Grid>
                    </StackPanel>
                </Border>
            </Grid>

            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="12"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>

                <!-- ══ LEFT COLUMN ══ -->
                <StackPanel Grid.Column="0">

                    <!-- IDENTITY -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="IDENTITY" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="120"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0" Grid.Column="0" Text="Hostname"        Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0" Grid.Column="1" x:Name="TxtHostname"   Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="1" Grid.ColumnSpan="2"                    Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="2" Grid.Column="0" Text="Logged In User"  Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2" Grid.Column="1" x:Name="TxtUser"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3" Grid.ColumnSpan="2"                    Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="4" Grid.Column="0" Text="Domain / Join"    Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtDomain"      Style="{StaticResource RowValue}" TextWrapping="Wrap"/>
                                <Rectangle Grid.Row="5" Grid.ColumnSpan="2"                    Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="6" Grid.Column="0" Text="Timezone"         Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6" Grid.Column="1" x:Name="TxtTimezone"    Style="{StaticResource RowValue}"/>
                            </Grid>
                        </StackPanel>
                    </Border>

                    <!-- HARDWARE -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="HARDWARE" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="140"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0"  Grid.Column="0" Text="Manufacturer / Model" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0"  Grid.Column="1" x:Name="TxtModel"           Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="1"  Grid.ColumnSpan="2"                         Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="2"  Grid.Column="0" Text="Serial Number"        Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2"  Grid.Column="1" x:Name="TxtSerial"          Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3"  Grid.ColumnSpan="2"                         Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="4"  Grid.Column="0" Text="BIOS Version"         Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4"  Grid.Column="1" x:Name="TxtBios"            Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="5"  Grid.ColumnSpan="2"                         Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="6"  Grid.Column="0" Text="BIOS Date"            Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6"  Grid.Column="1" x:Name="TxtBiosDate"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="7"  Grid.ColumnSpan="2"                         Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="8"  Grid.Column="0" Text="Boot Mode"            Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="8"  Grid.Column="1" x:Name="TxtBootMode"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="9"  Grid.ColumnSpan="2"                         Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="10" Grid.Column="0" Text="TPM"                  Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="10" Grid.Column="1" x:Name="TxtTpm"             Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="11" Grid.ColumnSpan="2" x:Name="WarrantyDivider" Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="12" Grid.Column="0" x:Name="WarrantyLabel" Text="Warranty" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="12" Grid.Column="1" x:Name="WarrantyLine" Style="{StaticResource RowValue}">
                                    <Hyperlink Click="Warranty_Click" Foreground="#89B4FA">Look up online</Hyperlink>
                                </TextBlock>
                            </Grid>
                        </StackPanel>
                    </Border>

                    <!-- OPERATING SYSTEM -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="OPERATING SYSTEM" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="120"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0" Grid.Column="0" Text="OS Version"      Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0" Grid.Column="1" x:Name="TxtOs"          Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="1" Grid.ColumnSpan="2"                    Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="2" Grid.Column="0" Text="Kernel Version"   Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2" Grid.Column="1" x:Name="TxtKernel"       Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3" Grid.ColumnSpan="2"                    Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="4" Grid.Column="0" Text="Architecture"     Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtArch"         Style="{StaticResource RowValue}"/>
                            </Grid>
                        </StackPanel>
                    </Border>

                    <!-- STORAGE -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <DockPanel>
                                <Button x:Name="BtnSmart" DockPanel.Dock="Right" Content="SMART details"
                                        Style="{StaticResource ActionButton}" Padding="10,4" FontSize="11"
                                        Click="Smart_Click"/>
                                <TextBlock Text="STORAGE" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                            </DockPanel>

                            <ItemsControl x:Name="IcDisks">
                                <ItemsControl.ItemTemplate>
                                    <DataTemplate>
                                        <StackPanel Margin="0,4">
                                            <DockPanel>
                                                <Ellipse Style="{StaticResource HealthDot}" DockPanel.Dock="Left"/>
                                                <TextBlock DockPanel.Dock="Right" VerticalAlignment="Center"
                                                           Foreground="#9399B2" FontSize="12" MinWidth="80" TextAlignment="Right"
                                                           Text="{Binding SizeGb, StringFormat={}{0:F0} GB}"/>
                                                <TextBlock DockPanel.Dock="Right" VerticalAlignment="Center"
                                                           Foreground="#9399B2" FontSize="12" Margin="0,0,16,0"
                                                           Text="{Binding Type}"/>
                                                <TextBlock Text="{Binding Model}" Foreground="#CDD6F4" FontSize="12"
                                                           VerticalAlignment="Center" TextTrimming="CharacterEllipsis"/>
                                            </DockPanel>
                                            <TextBlock Text="{Binding SmartText}" Foreground="#6C7086" FontSize="11"
                                                       Margin="19,2,0,0"
                                                       Visibility="{Binding HasSmart, Converter={StaticResource BoolToVis}}"/>
                                        </StackPanel>
                                    </DataTemplate>
                                </ItemsControl.ItemTemplate>
                            </ItemsControl>

                            <Rectangle Style="{StaticResource RowDivider}" Margin="0,8"/>

                            <ItemsControl x:Name="IcVolumes">
                                <ItemsControl.ItemTemplate>
                                    <DataTemplate>
                                        <DockPanel Margin="0,6">
                                            <TextBlock DockPanel.Dock="Left" Text="{Binding Letter}" Width="34"
                                                       Foreground="#CDD6F4" FontWeight="SemiBold" VerticalAlignment="Center"/>
                                            <TextBlock DockPanel.Dock="Right" VerticalAlignment="Center"
                                                       Foreground="#6C7086" FontSize="11"
                                                       Text="{Binding FreeGb, StringFormat={}{0:F0} GB free}"/>
                                            <StackPanel VerticalAlignment="Center">
                                                <TextBlock FontSize="12">
                                                    <Run Text="{Binding Label, Mode=OneWay}" Foreground="#CDD6F4"/>
                                                    <Run Text="  " />
                                                    <Run Text="{Binding TotalGb, StringFormat={}{0:F0} GB, Mode=OneWay}" Foreground="#6C7086"/>
                                                </TextBlock>
                                                <ProgressBar Style="{StaticResource UsageBar}" Maximum="100"
                                                             Value="{Binding UsedPct, Mode=OneWay}" Margin="0,4,8,0"/>
                                            </StackPanel>
                                        </DockPanel>
                                    </DataTemplate>
                                </ItemsControl.ItemTemplate>
                            </ItemsControl>

                        </StackPanel>
                    </Border>

                    <!-- MEMORY -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="MEMORY" Style="{StaticResource CardTitle}"/>
                            <TextBlock x:Name="TxtMemSummary" Foreground="#CDD6F4" FontSize="12" Margin="0,0,0,8"/>
                            <StackPanel Grid.IsSharedSizeScope="True">
                                <!-- Header row (shares column widths with the item rows) -->
                                <Grid Margin="0,0,0,2">
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition SharedSizeGroup="MemSlot"/>
                                        <ColumnDefinition SharedSizeGroup="MemSize"/>
                                        <ColumnDefinition SharedSizeGroup="MemSpeed"/>
                                        <ColumnDefinition Width="*"/>
                                    </Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="Slot"  Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,12,0"/>
                                    <TextBlock Grid.Column="1" Text="Size"  Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,12,0"/>
                                    <TextBlock Grid.Column="2" Text="Speed" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,12,0"/>
                                    <TextBlock Grid.Column="3" Text="Maker" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                                </Grid>
                                <ItemsControl x:Name="IcMemory">
                                    <ItemsControl.ItemTemplate>
                                        <DataTemplate>
                                            <Grid Margin="0,3">
                                                <Grid.ColumnDefinitions>
                                                    <ColumnDefinition SharedSizeGroup="MemSlot"/>
                                                    <ColumnDefinition SharedSizeGroup="MemSize"/>
                                                    <ColumnDefinition SharedSizeGroup="MemSpeed"/>
                                                    <ColumnDefinition Width="*"/>
                                                </Grid.ColumnDefinitions>
                                                <TextBlock Grid.Column="0" Text="{Binding SlotText}"  Foreground="#9399B2" FontSize="12" Margin="0,0,12,0"/>
                                                <TextBlock Grid.Column="1" Text="{Binding SizeText}"  Foreground="#CDD6F4" FontSize="12" Margin="0,0,12,0"/>
                                                <TextBlock Grid.Column="2" Text="{Binding SpeedText}" Foreground="#9399B2" FontSize="12" Margin="0,0,12,0"/>
                                                <TextBlock Grid.Column="3" Text="{Binding MakerText}" Foreground="#6C7086" FontSize="12" TextTrimming="CharacterEllipsis"/>
                                            </Grid>
                                        </DataTemplate>
                                    </ItemsControl.ItemTemplate>
                                </ItemsControl>
                            </StackPanel>
                        </StackPanel>
                    </Border>

                    <!-- NETWORK -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="NETWORK" Style="{StaticResource CardTitle}"/>
                            <TextBlock x:Name="TxtNetName" FontSize="12" FontWeight="SemiBold"
                                       Foreground="#CDD6F4" Margin="0,0,0,8"/>
                            <Grid x:Name="NetGrid">
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="120"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0"  Grid.Column="0" Text="Type"        Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0"  Grid.Column="1" x:Name="TxtNetType" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="1"  Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="2"  Grid.Column="0" Text="IP Address"  Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2"  Grid.Column="1" x:Name="TxtNetIp"  Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3"  Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="4"  Grid.Column="0" Text="Subnet Mask" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4"  Grid.Column="1" x:Name="TxtNetMask" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="5"  Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="6"  Grid.Column="0" Text="Gateway"     Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6"  Grid.Column="1" x:Name="TxtNetGw"  Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="7"  Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="8"  Grid.Column="0" Text="DNS Servers" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="8"  Grid.Column="1" x:Name="TxtNetDns" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="9"  Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="10" Grid.Column="0" Text="MAC Address" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="10" Grid.Column="1" x:Name="TxtNetMac" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="11" Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="12" Grid.Column="0" Text="Assignment"  Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="12" Grid.Column="1" x:Name="TxtNetDhcp" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="13" Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="14" Grid.Column="0" Text="DHCP Server" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="14" Grid.Column="1" x:Name="TxtNetDhcpServer" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="15" Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="16" Grid.Column="0" Text="Lease Expires" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="16" Grid.Column="1" x:Name="TxtNetLease" Style="{StaticResource RowValue}"/>
                            </Grid>
                            <TextBlock x:Name="TxtNoNet" Text="No primary adapter detected."
                                       Foreground="#6C7086" FontSize="12" Visibility="Collapsed"/>
                        </StackPanel>
                    </Border>

                    <!-- LOCAL ACCOUNTS -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="LOCAL ACCOUNTS" Style="{StaticResource CardTitle}"/>
                            <ItemsControl x:Name="IcAccounts">
                                <ItemsControl.ItemTemplate>
                                    <DataTemplate>
                                        <DockPanel Margin="0,4">
                                            <Ellipse DockPanel.Dock="Left" Width="9" Height="9" VerticalAlignment="Center" Margin="0,0,10,0">
                                                <Ellipse.Style>
                                                    <Style TargetType="Ellipse">
                                                        <Setter Property="Fill" Value="#89B4FA"/>
                                                        <Style.Triggers>
                                                            <DataTrigger Binding="{Binding IsAdmin}" Value="True">
                                                                <Setter Property="Fill" Value="#F9E2AF"/>
                                                            </DataTrigger>
                                                            <DataTrigger Binding="{Binding Disabled}" Value="True">
                                                                <Setter Property="Fill" Value="#6C7086"/>
                                                            </DataTrigger>
                                                        </Style.Triggers>
                                                    </Style>
                                                </Ellipse.Style>
                                            </Ellipse>
                                            <StackPanel>
                                                <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12" FontWeight="SemiBold"/>
                                                <TextBlock Text="{Binding Detail}" Foreground="#6C7086" FontSize="11" Margin="0,1,0,0"/>
                                            </StackPanel>
                                        </DockPanel>
                                    </DataTemplate>
                                </ItemsControl.ItemTemplate>
                            </ItemsControl>
                        </StackPanel>
                    </Border>

                </StackPanel>

                <!-- ══ RIGHT COLUMN ══ -->
                <StackPanel Grid.Column="2">

                    <!-- PERFORMANCE -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="PERFORMANCE" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="110"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>

                                <!-- CPU -->
                                <TextBlock Grid.Row="0" Grid.Column="0" Text="Processor"  Style="{StaticResource RowLabel}"/>
                                <StackPanel Grid.Row="0" Grid.Column="1" Margin="0,5,0,5">
                                    <TextBlock x:Name="TxtCpu" Foreground="#CDD6F4" FontSize="12" TextWrapping="Wrap"/>
                                    <TextBlock x:Name="TxtCpuClock" Foreground="#9399B2" FontSize="11" Margin="0,2,0,0"/>
                                </StackPanel>
                                <Rectangle Grid.Row="1" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>

                                <!-- RAM -->
                                <TextBlock Grid.Row="2" Grid.Column="0" Text="Memory"     Style="{StaticResource RowLabel}"/>
                                <StackPanel Grid.Row="2" Grid.Column="1" Margin="0,5,0,5">
                                    <TextBlock x:Name="TxtRam" Foreground="#CDD6F4" FontSize="12"/>
                                    <ProgressBar x:Name="BarRam" Style="{StaticResource UsageBar}" Margin="0,4,0,0"/>
                                </StackPanel>
                                <Rectangle Grid.Row="3" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>

                                <!-- Uptime -->
                                <TextBlock Grid.Row="4" Grid.Column="0" Text="Uptime"     Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtUptime" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="5" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>

                                <!-- Last Reboot -->
                                <TextBlock Grid.Row="6" Grid.Column="0" Text="Last Reboot" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6" Grid.Column="1" x:Name="TxtLastBoot" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="7" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>

                                <!-- Page file -->
                                <TextBlock Grid.Row="8" Grid.Column="0" Text="Page file"   Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="8" Grid.Column="1" x:Name="TxtPageFile" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="9" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>

                                <!-- Proxy -->
                                <TextBlock Grid.Row="10" Grid.Column="0" Text="Proxy"      Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="10" Grid.Column="1" x:Name="TxtProxy"  Style="{StaticResource RowValue}"/>
                            </Grid>
                        </StackPanel>
                    </Border>

                    <!-- GRAPHICS & DISPLAY -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="GRAPHICS &amp; DISPLAY" Style="{StaticResource CardTitle}"/>
                            <ItemsControl x:Name="IcGpus">
                                <ItemsControl.ItemTemplate>
                                    <DataTemplate>
                                        <StackPanel Margin="0,5">
                                            <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12" FontWeight="SemiBold"/>
                                            <TextBlock Foreground="#6C7086" FontSize="11" Margin="0,2,0,0">
                                                <Run Text="{Binding Resolution, Mode=OneWay}"/>
                                                <Run Text="   ·   driver "/>
                                                <Run Text="{Binding Driver, Mode=OneWay}"/>
                                                <Run Text="  ("/>
                                                <Run Text="{Binding DriverDate, Mode=OneWay}"/>
                                                <Run Text=")"/>
                                            </TextBlock>
                                        </StackPanel>
                                    </DataTemplate>
                                </ItemsControl.ItemTemplate>
                            </ItemsControl>

                            <TextBlock x:Name="TxtVram" Foreground="#9399B2" FontSize="11" Margin="0,4,0,0"
                                       Visibility="Collapsed"/>

                            <!-- Connected monitors (EDID) — Make / Model / Size / Year / Serial columns -->
                            <StackPanel x:Name="MonitorsSection" Visibility="Collapsed" Grid.IsSharedSizeScope="True">
                                <Rectangle Style="{StaticResource RowDivider}" Margin="0,8"/>
                                <TextBlock Text="MONITORS" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                                <!-- Header row (shares column widths with the item rows via SharedSizeGroup) -->
                                <Grid Margin="0,0,0,2">
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition SharedSizeGroup="MonMake"/>
                                        <ColumnDefinition Width="*"/>
                                        <ColumnDefinition SharedSizeGroup="MonSize"/>
                                        <ColumnDefinition SharedSizeGroup="MonYear"/>
                                        <ColumnDefinition SharedSizeGroup="MonSerial"/>
                                    </Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="Make"   Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,10,0"/>
                                    <TextBlock Grid.Column="1" Text="Model"  Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,10,0"/>
                                    <TextBlock Grid.Column="2" Text="Size"   Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,10,0"/>
                                    <TextBlock Grid.Column="3" Text="Year"   Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,10,0"/>
                                    <TextBlock Grid.Column="4" Text="Serial" Foreground="#6C7086" FontSize="10" FontWeight="Bold"/>
                                </Grid>
                                <ItemsControl x:Name="IcMonitors">
                                    <ItemsControl.ItemTemplate>
                                        <DataTemplate>
                                            <Grid Margin="0,3">
                                                <Grid.ColumnDefinitions>
                                                    <ColumnDefinition SharedSizeGroup="MonMake"/>
                                                    <ColumnDefinition Width="*"/>
                                                    <ColumnDefinition SharedSizeGroup="MonSize"/>
                                                    <ColumnDefinition SharedSizeGroup="MonYear"/>
                                                    <ColumnDefinition SharedSizeGroup="MonSerial"/>
                                                </Grid.ColumnDefinitions>
                                                <TextBlock Grid.Column="0" Text="{Binding MakeText}"   Foreground="#CDD6F4" FontSize="12" Margin="0,0,10,0"/>
                                                <TextBlock Grid.Column="1" Text="{Binding ModelText}"  Foreground="#CDD6F4" FontSize="12" Margin="0,0,10,0" TextTrimming="CharacterEllipsis"/>
                                                <TextBlock Grid.Column="2" Text="{Binding SizeText}"   Foreground="#9399B2" FontSize="12" Margin="0,0,10,0"/>
                                                <TextBlock Grid.Column="3" Text="{Binding YearText}"   Foreground="#9399B2" FontSize="12" Margin="0,0,10,0"/>
                                                <TextBlock Grid.Column="4" Text="{Binding SerialText}" Foreground="#9399B2" FontSize="12" TextTrimming="CharacterEllipsis"/>
                                            </Grid>
                                        </DataTemplate>
                                    </ItemsControl.ItemTemplate>
                                </ItemsControl>
                            </StackPanel>
                        </StackPanel>
                    </Border>

                    <!-- SECURITY -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="SECURITY" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="110"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>

                                <TextBlock Grid.Row="0"  Grid.Column="0" Text="Antivirus"    Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0"  Grid.Column="1" x:Name="TxtAv"       Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="1"  Grid.ColumnSpan="2"                  Style="{StaticResource RowDivider}"/>

                                <TextBlock Grid.Row="2"  Grid.Column="0" Text="Firewall"      Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2"  Grid.Column="1" x:Name="TxtFw"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3"  Grid.ColumnSpan="2"                  Style="{StaticResource RowDivider}"/>

                                <TextBlock Grid.Row="4"  Grid.Column="0" Text="BitLocker"     Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4"  Grid.Column="1" x:Name="TxtBl"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="5"  Grid.ColumnSpan="2"                  Style="{StaticResource RowDivider}"/>

                                <TextBlock Grid.Row="6"  Grid.Column="0" Text="Activation"    Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6"  Grid.Column="1" x:Name="TxtAct"       Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="7"  Grid.ColumnSpan="2"                  Style="{StaticResource RowDivider}"/>

                                <TextBlock Grid.Row="8"  Grid.Column="0" Text="Secure Boot"   Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="8"  Grid.Column="1" x:Name="TxtSb"        Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="9"  Grid.ColumnSpan="2"                  Style="{StaticResource RowDivider}"/>

                                <TextBlock Grid.Row="10" Grid.Column="0" Text="UAC"            Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="10" Grid.Column="1" x:Name="TxtUac"       Style="{StaticResource RowValue}"/>
                            </Grid>
                        </StackPanel>
                    </Border>

                    <!-- BATTERY (hidden on desktops) -->
                    <Border x:Name="BatteryCard" Style="{StaticResource Card}" Visibility="Collapsed">
                        <StackPanel>
                            <TextBlock Text="BATTERY" Style="{StaticResource CardTitle}"/>
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="110"/>
                                    <ColumnDefinition Width="*"/>
                                </Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0" Grid.Column="0" Text="Charge"     Style="{StaticResource RowLabel}"/>
                                <StackPanel Grid.Row="0" Grid.Column="1" Margin="0,5,0,5">
                                    <TextBlock x:Name="TxtBattery" Foreground="#CDD6F4" FontSize="12"/>
                                    <ProgressBar x:Name="BarBattery" Style="{StaticResource UsageBar}" Margin="0,4,0,0"/>
                                </StackPanel>
                                <Rectangle Grid.Row="1" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="2" Grid.Column="0" Text="Status"     Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2" Grid.Column="1" x:Name="TxtBattPower" Style="{StaticResource RowValue}"/>
                                <Rectangle Grid.Row="3" Grid.ColumnSpan="2"               Style="{StaticResource RowDivider}"/>
                                <TextBlock Grid.Row="4" Grid.Column="0" Text="Voltage"    Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtBattVoltage" Style="{StaticResource RowValue}"/>
                            </Grid>

                            <!-- Battery health (wear) — shown only when capacity is readable -->
                            <StackPanel x:Name="BattHealthSection" Visibility="Collapsed">
                                <Rectangle Style="{StaticResource RowDivider}" Margin="0,6"/>
                                <Grid>
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition Width="110"/>
                                        <ColumnDefinition Width="*"/>
                                    </Grid.ColumnDefinitions>
                                    <Grid.RowDefinitions>
                                        <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    </Grid.RowDefinitions>
                                    <TextBlock Grid.Row="0" Grid.Column="0" Text="Wear"        Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Row="0" Grid.Column="1" x:Name="TxtWear"   Style="{StaticResource RowValue}"/>
                                    <Rectangle Grid.Row="1" Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                    <TextBlock Grid.Row="2" Grid.Column="0" Text="Design"      Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Row="2" Grid.Column="1" x:Name="TxtBattDesign" Style="{StaticResource RowValue}"/>
                                    <Rectangle Grid.Row="3" Grid.ColumnSpan="2"                Style="{StaticResource RowDivider}"/>
                                    <TextBlock Grid.Row="4" Grid.Column="0" Text="Full charge" Style="{StaticResource RowLabel}"/>
                                    <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtBattFull"   Style="{StaticResource RowValue}"/>
                                </Grid>
                            </StackPanel>
                        </StackPanel>
                    </Border>

                    <!-- PRINTERS (moved up to fill the right-column gap) -->
                    <Border Style="{StaticResource Card}">
                        <StackPanel>
                            <TextBlock Text="PRINTERS" Style="{StaticResource CardTitle}"/>
                            <ItemsControl x:Name="IcPrinters">
                                <ItemsControl.ItemTemplate>
                                    <DataTemplate>
                                        <StackPanel Margin="0,4">
                                            <TextBlock Text="{Binding Display}" Foreground="#CDD6F4" FontSize="12"/>
                                            <TextBlock Text="{Binding Detail}" Foreground="#6C7086" FontSize="11"
                                                       Margin="0,1,0,0" TextTrimming="CharacterEllipsis"/>
                                        </StackPanel>
                                    </DataTemplate>
                                </ItemsControl.ItemTemplate>
                            </ItemsControl>
                            <TextBlock x:Name="TxtNoPrinters" Foreground="#6C7086" FontSize="12"
                                       Text="No printers installed." Visibility="Collapsed"/>
                        </StackPanel>
                    </Border>

                </StackPanel>
            </Grid>


            </StackPanel>
        </ScrollViewer>
    </Grid>
</UserControl>
```
