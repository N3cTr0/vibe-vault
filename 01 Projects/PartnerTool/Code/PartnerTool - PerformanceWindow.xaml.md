---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PerformanceWindow.xaml
---

# PartnerTool\PerformanceWindow.xaml

```xml
<Window x:Class="PartnerTool.PerformanceWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Performance" Width="1280" Height="680" MinWidth="720" MinHeight="480"
        Background="#1E1E2E" WindowStartupLocation="CenterOwner">

    <Window.Resources>
        <!-- Each rail tile is a visible boxed card; the selected one gets a bright border (set in
             code via the Button's BorderBrush/Thickness, template-bound), and hovering lightens the
             fill. Buttons (not Borders) so the tiles are keyboard- and automation-accessible. -->
        <Style x:Key="RailTile" TargetType="Button">
            <Setter Property="Margin" Value="0,0,0,8"/>
            <Setter Property="Background" Value="#1E1E2E"/>
            <Setter Property="BorderBrush" Value="#313244"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="Bd" CornerRadius="6" Padding="8"
                                Background="{TemplateBinding Background}"
                                BorderBrush="{TemplateBinding BorderBrush}"
                                BorderThickness="{TemplateBinding BorderThickness}">
                            <ContentPresenter HorizontalAlignment="Stretch" VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="Bd" Property="Background" Value="#242536"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </Window.Resources>

    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="232"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>

        <!-- ── LEFT RAIL ── -->
        <ScrollViewer Grid.Column="0" Background="#181825" VerticalScrollBarVisibility="Auto">
            <StackPanel Margin="10,12">
                <Button x:Name="TileCpu" Tag="cpu" Style="{StaticResource RailTile}" Click="Tile_Click">
                    <DockPanel>
                        <Border DockPanel.Dock="Left" Width="58" Height="36" Background="#11111B" CornerRadius="4" ClipToBounds="True" Margin="0,0,10,0">
                            <Polyline x:Name="CpuMini" Stroke="#89B4FA" StrokeThickness="1.2"/>
                        </Border>
                        <StackPanel VerticalAlignment="Center">
                            <TextBlock Text="CPU" Foreground="#CDD6F4" FontSize="13" FontWeight="SemiBold"/>
                            <TextBlock x:Name="TxtCpuRail" Foreground="#9399B2" FontSize="11" Text="—"/>
                        </StackPanel>
                    </DockPanel>
                </Button>
                <Button x:Name="TileMem" Tag="mem" Style="{StaticResource RailTile}" Click="Tile_Click">
                    <DockPanel>
                        <Border DockPanel.Dock="Left" Width="58" Height="36" Background="#11111B" CornerRadius="4" ClipToBounds="True" Margin="0,0,10,0">
                            <Polyline x:Name="MemMini" Stroke="#A6E3A1" StrokeThickness="1.2"/>
                        </Border>
                        <StackPanel VerticalAlignment="Center">
                            <TextBlock Text="Memory" Foreground="#CDD6F4" FontSize="13" FontWeight="SemiBold"/>
                            <TextBlock x:Name="TxtMemRail" Foreground="#9399B2" FontSize="11" Text="—"/>
                        </StackPanel>
                    </DockPanel>
                </Button>
                <Button x:Name="TileDisk" Tag="disk" Style="{StaticResource RailTile}" Click="Tile_Click">
                    <DockPanel>
                        <Border DockPanel.Dock="Left" Width="58" Height="36" Background="#11111B" CornerRadius="4" ClipToBounds="True" Margin="0,0,10,0">
                            <Polyline x:Name="DiskMini" Stroke="#F9E2AF" StrokeThickness="1.2"/>
                        </Border>
                        <StackPanel VerticalAlignment="Center">
                            <TextBlock Text="Disk" Foreground="#CDD6F4" FontSize="13" FontWeight="SemiBold"/>
                            <TextBlock x:Name="TxtDiskRail" Foreground="#9399B2" FontSize="11" Text="—"/>
                        </StackPanel>
                    </DockPanel>
                </Button>
                <Button x:Name="TileNet" Tag="net" Style="{StaticResource RailTile}" Click="Tile_Click">
                    <DockPanel>
                        <Border DockPanel.Dock="Left" Width="58" Height="36" Background="#11111B" CornerRadius="4" ClipToBounds="True" Margin="0,0,10,0">
                            <Polyline x:Name="NetMini" Stroke="#CBA6F7" StrokeThickness="1.2"/>
                        </Border>
                        <StackPanel VerticalAlignment="Center">
                            <TextBlock Text="Network" Foreground="#CDD6F4" FontSize="13" FontWeight="SemiBold"/>
                            <TextBlock x:Name="TxtNetRail" Foreground="#9399B2" FontSize="11" Text="—"/>
                        </StackPanel>
                    </DockPanel>
                </Button>

                <!-- Processes lives at the bottom, under the four live-graph resources -->
                <Border Height="1" Background="#313244" Margin="2,6,2,10"/>
                <Button x:Name="TileProc" Tag="proc" Style="{StaticResource RailTile}" Click="Tile_Click">
                    <DockPanel>
                        <Border DockPanel.Dock="Left" Width="58" Height="36" Background="#11111B" CornerRadius="4" Margin="0,0,10,0">
                            <TextBlock Text="▤" Foreground="#89B4FA" FontSize="18" HorizontalAlignment="Center" VerticalAlignment="Center"/>
                        </Border>
                        <StackPanel VerticalAlignment="Center">
                            <TextBlock Text="Processes" Foreground="#CDD6F4" FontSize="13" FontWeight="SemiBold"/>
                            <TextBlock x:Name="TxtProcRail" Foreground="#9399B2" FontSize="11" Text="—"/>
                        </StackPanel>
                    </DockPanel>
                </Button>
            </StackPanel>
        </ScrollViewer>

        <!-- ── RIGHT DETAIL ── -->
        <Grid Grid.Column="1" Margin="22,18,22,18">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="*"/>
                <RowDefinition Height="Auto"/>
            </Grid.RowDefinitions>

            <DockPanel Grid.Row="0">
                <TextBlock DockPanel.Dock="Right" x:Name="TxtResSub" Foreground="#9399B2" FontSize="14" VerticalAlignment="Bottom"/>
                <TextBlock x:Name="TxtResTitle" Foreground="#CDD6F4" FontSize="26" FontWeight="SemiBold"/>
            </DockPanel>

            <Border Grid.Row="1" x:Name="GraphCard" Background="#11111B" CornerRadius="6" ClipToBounds="True" Margin="0,12">
                <Grid>
                    <!-- single overall graph (Memory / Disk / Network) -->
                    <Grid x:Name="SingleGraph">
                        <TextBlock x:Name="TxtGraphMax" Text="100%" Foreground="#6C7086" FontSize="10"
                                   HorizontalAlignment="Right" VerticalAlignment="Top" Margin="0,4,6,0"/>
                        <Border x:Name="BigPlot" ClipToBounds="True">
                            <Polyline x:Name="BigLine" Stroke="#89B4FA" StrokeThickness="1.6"/>
                        </Border>
                    </Grid>
                    <!-- per-logical-processor grid (CPU) — built in code -->
                    <Grid x:Name="CorePanel" Visibility="Collapsed">
                        <Grid.RowDefinitions><RowDefinition Height="Auto"/><RowDefinition Height="*"/></Grid.RowDefinitions>
                        <TextBlock Grid.Row="0" Text="Logical processors — % utilization" Foreground="#6C7086"
                                   FontSize="10" Margin="6,4,0,2"/>
                        <UniformGrid Grid.Row="1" x:Name="CoreGrid" Margin="4"/>
                    </Grid>
                </Grid>
            </Border>

            <!-- Processes view (Task-Manager-style list) replaces the graph + stats when selected -->
            <ContentControl Grid.Row="1" Grid.RowSpan="2" x:Name="ProcHost" Visibility="Collapsed" Margin="0,12,0,0"/>

            <ItemsControl Grid.Row="2" x:Name="IcStats">
                <ItemsControl.ItemsPanel>
                    <ItemsPanelTemplate><UniformGrid Columns="2"/></ItemsPanelTemplate>
                </ItemsControl.ItemsPanel>
                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <DockPanel Margin="0,4,18,4">
                            <TextBlock DockPanel.Dock="Left" Width="118" Text="{Binding Label}" Foreground="#6C7086" FontSize="12"/>
                            <TextBlock Text="{Binding Value}" Foreground="#CDD6F4" FontSize="12" FontWeight="SemiBold" TextTrimming="CharacterEllipsis"/>
                        </DockPanel>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>
        </Grid>
    </Grid>
</Window>
```
