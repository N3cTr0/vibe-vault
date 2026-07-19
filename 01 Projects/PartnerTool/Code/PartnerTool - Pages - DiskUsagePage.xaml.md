---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\DiskUsagePage.xaml
---

# PartnerTool\Pages\DiskUsagePage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.DiskUsagePage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <UserControl.Resources>
        <!-- Dark column header. Click to sort (handled in Header_Click). -->
        <Style TargetType="GridViewColumnHeader">
            <Setter Property="Foreground" Value="#9399B2"/>
            <Setter Property="FontSize" Value="11"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="HorizontalContentAlignment" Value="Left"/>
            <Setter Property="Cursor" Value="Hand"/>
            <EventSetter Event="Click" Handler="Header_Click"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="GridViewColumnHeader">
                        <Border x:Name="hbg" Background="#181825" BorderBrush="#313244" BorderThickness="0,0,1,1" Padding="8,6">
                            <ContentPresenter HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="hbg" Property="Background" Value="#242536"/>
                                <Setter Property="Foreground" Value="#CDD6F4"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
        <Style x:Key="RightHeader" TargetType="GridViewColumnHeader" BasedOn="{StaticResource {x:Type GridViewColumnHeader}}">
            <Setter Property="HorizontalContentAlignment" Value="Right"/>
        </Style>

        <!-- Row: hover / select highlight, no default chrome -->
        <Style x:Key="RowStyle" TargetType="ListViewItem">
            <Setter Property="Padding" Value="0"/>
            <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
            <Setter Property="AutomationProperties.Name" Value="{Binding AutomationName}"/>
            <EventSetter Event="MouseDoubleClick" Handler="Row_Click"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ListViewItem">
                        <Border x:Name="bg" Background="Transparent" Padding="0,2">
                            <GridViewRowPresenter Content="{TemplateBinding Content}"
                                                  Columns="{TemplateBinding GridView.ColumnCollection}"/>
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

        <!-- Folder / file / up icon -->
        <Style x:Key="RowIcon" TargetType="TextBlock">
            <Setter Property="FontFamily" Value="Segoe MDL2 Assets"/>
            <Setter Property="FontSize" Value="13"/>
            <Setter Property="VerticalAlignment" Value="Center"/>
            <Setter Property="Margin" Value="0,0,6,0"/>
            <Setter Property="Text" Value="&#xE8A5;"/>
            <Setter Property="Foreground" Value="#6C7086"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding IsDir}" Value="True">
                    <Setter Property="Text" Value="&#xE8B7;"/>
                    <Setter Property="Foreground" Value="#89B4FA"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding IsParent}" Value="True">
                    <Setter Property="Text" Value="&#xE74A;"/>
                    <Setter Property="Foreground" Value="#9399B2"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>

        <!-- Name text: bright normally, dimmed for hidden/system, muted for the ".." row -->
        <Style x:Key="RowName" TargetType="TextBlock">
            <Setter Property="VerticalAlignment" Value="Center"/>
            <Setter Property="TextTrimming" Value="CharacterEllipsis"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding Hidden}" Value="True">
                    <Setter Property="Foreground" Value="#6C7086"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding IsParent}" Value="True">
                    <Setter Property="Foreground" Value="#9399B2"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </UserControl.Resources>

    <Grid Background="#1E1E2E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>   <!-- header + drive picker + summary -->
            <RowDefinition Height="Auto"/>   <!-- nav bar -->
            <RowDefinition Height="Auto"/>   <!-- status + busy -->
            <RowDefinition Height="*"/>      <!-- results -->
        </Grid.RowDefinitions>

        <StackPanel Grid.Row="0" Margin="20,16,20,4">
            <TextBlock Text="DISK USAGE" Style="{StaticResource CardTitle}"/>
            <TextBlock Text="Find what's using space, TreeSize/WizTree-style. Pick a drive — NTFS drives get a ⚡ fast scan (reads the Master File Table directly); others fall back to a folder walk. Sizes are 'size on disk' (so cloud-only OneDrive files count as ~0). Double-click a folder to browse into it, or the '..' row to go up. Click any column header to sort (click again to reverse)."
                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,8"/>
            <WrapPanel x:Name="PnlDrives"/>
            <TextBlock x:Name="TxtSummary" Foreground="#9399B2" FontSize="11" Margin="0,6,0,0" TextWrapping="Wrap"/>
        </StackPanel>

        <DockPanel Grid.Row="1" Margin="20,6,20,4">
            <Button x:Name="BtnRescan" DockPanel.Dock="Left" Content="Rescan" Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="Rescan_Click" IsEnabled="False"/>
            <Button x:Name="BtnOpen"   DockPanel.Dock="Left" Content="Open in Explorer" Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="Open_Click" IsEnabled="False"/>
            <CheckBox x:Name="ChkHidden" DockPanel.Dock="Right" Content="Show hidden items" IsChecked="True"
                      Foreground="#9399B2" FontSize="11" VerticalAlignment="Center" Margin="8,0,0,0"
                      Click="Hidden_Changed"/>
            <TextBlock x:Name="TxtPath" Foreground="#CDD6F4" FontSize="12" VerticalAlignment="Center"
                       TextTrimming="CharacterEllipsis" Text="Select a drive above to begin."/>
        </DockPanel>

        <StackPanel Grid.Row="2" Margin="20,0,20,6">
            <TextBlock x:Name="TxtStatus" Foreground="#6C7086" FontSize="11"/>
            <ProgressBar x:Name="Busy" Height="3" IsIndeterminate="True" Margin="0,4,0,0"
                         Visibility="Collapsed" Background="#313244" Foreground="#89B4FA" BorderThickness="0"/>
        </StackPanel>

        <ListView x:Name="LstEntries" Grid.Row="3" Margin="20,0,20,16" Background="#1E1E2E" BorderThickness="0"
                  Foreground="#CDD6F4" ItemContainerStyle="{StaticResource RowStyle}"
                  VirtualizingPanel.IsVirtualizing="True" VirtualizingPanel.VirtualizationMode="Recycling"
                  ScrollViewer.HorizontalScrollBarVisibility="Disabled">
            <ListView.View>
                <GridView>
                    <!-- Name -->
                    <GridViewColumn Header="Name" Width="330">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <StackPanel Orientation="Horizontal">
                                    <TextBlock Style="{StaticResource RowIcon}"/>
                                    <TextBlock Text="{Binding Display}" Style="{StaticResource RowName}"/>
                                </StackPanel>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>

                    <!-- % of Parent -->
                    <GridViewColumn Header="% of Parent" Width="170">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <DockPanel VerticalAlignment="Center" LastChildFill="True">
                                    <TextBlock DockPanel.Dock="Right" Text="{Binding PercentText}" Foreground="#9399B2"
                                               FontSize="10" Width="36" TextAlignment="Right" VerticalAlignment="Center"/>
                                    <ProgressBar Height="8" Minimum="0" Maximum="100" Value="{Binding Percent}" Margin="0,0,6,0"
                                                 Background="#313244" Foreground="#89B4FA" BorderThickness="0">
                                        <ProgressBar.Style>
                                            <Style TargetType="ProgressBar">
                                                <Style.Triggers>
                                                    <DataTrigger Binding="{Binding IsParent}" Value="True">
                                                        <Setter Property="Visibility" Value="Hidden"/>
                                                    </DataTrigger>
                                                </Style.Triggers>
                                            </Style>
                                        </ProgressBar.Style>
                                    </ProgressBar>
                                </DockPanel>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>

                    <!-- Size (on disk) -->
                    <GridViewColumn Header="Size" Width="90" HeaderContainerStyle="{StaticResource RightHeader}">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding SizeText}" Foreground="#CDD6F4" FontSize="12"
                                           HorizontalAlignment="Stretch" TextAlignment="Right"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>

                    <!-- Files (recursive) -->
                    <GridViewColumn Header="Files" Width="72" HeaderContainerStyle="{StaticResource RightHeader}">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding FilesText}" Foreground="#9399B2" FontSize="12"
                                           HorizontalAlignment="Stretch" TextAlignment="Right"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>

                    <!-- Modified -->
                    <GridViewColumn Header="Modified" Width="135">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding ModifiedText}" Foreground="#9399B2" FontSize="11" VerticalAlignment="Center"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>

                    <!-- Attributes -->
                    <GridViewColumn Header="Attr" Width="52">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding AttrText}" Foreground="#9399B2" FontSize="11" VerticalAlignment="Center"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>
                </GridView>
            </ListView.View>
        </ListView>
    </Grid>
</UserControl>
```
