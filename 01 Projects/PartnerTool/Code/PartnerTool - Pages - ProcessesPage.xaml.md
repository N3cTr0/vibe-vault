---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ProcessesPage.xaml
---

# PartnerTool\Pages\ProcessesPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.ProcessesPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:pt="clr-namespace:PartnerTool">

    <UserControl.Resources>
        <Style TargetType="ScrollViewer">
            <Setter Property="pt:ScrollChaining.Enabled" Value="True"/>
        </Style>
        <!-- Compact column-header sort button (matches the Manage page's SortBtn look) -->
        <Style x:Key="ColBtn" TargetType="Button">
            <Setter Property="Background" Value="Transparent"/>
            <Setter Property="Foreground" Value="#6C7086"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Padding" Value="0"/>
            <Setter Property="HorizontalContentAlignment" Value="Left"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <ContentPresenter HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}" VerticalAlignment="Center"/>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="Foreground" Value="#CDD6F4"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </UserControl.Resources>

    <Grid Background="#1E1E2E">
        <Border Style="{StaticResource Card}" Margin="20,16,20,16">
            <DockPanel>
                <DockPanel DockPanel.Dock="Top">
                    <Button x:Name="BtnPause" DockPanel.Dock="Right" Content="Pause"
                            Style="{StaticResource ActionButton}" Click="Pause_Click" Margin="8,0,0,0"/>
                    <TextBlock x:Name="TxtProcCount" DockPanel.Dock="Right" Foreground="#6C7086"
                               FontSize="11" VerticalAlignment="Center" Margin="0,0,10,0"/>
                    <StackPanel>
                        <TextBlock Text="PROCESSES" Style="{StaticResource CardTitle}"/>
                        <TextBlock Text="Live view of every running process — refreshes every 2 seconds. CPU is across all cores; Disk is that process's read+write rate. (Per-process network needs Windows' ETW tracing, so Task Manager's Network column isn't shown.)"
                                   FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                    </StackPanel>
                </DockPanel>

                <Border DockPanel.Dock="Top" Background="#45475A" CornerRadius="6" Padding="10,0" Margin="0,10,0,8">
                    <TextBox x:Name="TxtProcSearch" Foreground="#CDD6F4" FontSize="12"
                             BorderThickness="0" Height="30" VerticalContentAlignment="Center" TextChanged="ProcSearch_TextChanged">
                        <TextBox.Style>
                            <Style TargetType="TextBox">
                                <Style.Resources>
                                    <VisualBrush x:Key="ph" Stretch="None" AlignmentX="Left">
                                        <VisualBrush.Visual>
                                            <TextBlock Text="Search by name, PID, user, or description…" Foreground="#6C7086" FontSize="12"/>
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

                <!-- Column headers (click to sort, click again to flip) -->
                <Grid DockPanel.Dock="Top" Margin="0,0,0,4">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="2*"/>
                        <ColumnDefinition Width="56"/>
                        <ColumnDefinition Width="100"/>
                        <ColumnDefinition Width="110"/>
                        <ColumnDefinition Width="64"/>
                        <ColumnDefinition Width="84"/>
                        <ColumnDefinition Width="76"/>
                        <ColumnDefinition Width="62"/>
                        <ColumnDefinition Width="62"/>
                        <ColumnDefinition Width="48"/>
                        <ColumnDefinition Width="3*"/>
                        <ColumnDefinition Width="54"/>
                    </Grid.ColumnDefinitions>
                    <Button Grid.Column="0"  x:Name="ColName"    Content="Name"        Tag="name"    Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="1"  x:Name="ColPid"     Content="PID"         Tag="pid"     Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="2"  x:Name="ColStatus"  Content="Status"      Tag="status"  Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="3"  x:Name="ColUser"    Content="User"        Tag="user"    Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="4"  x:Name="ColCpu"     Content="CPU"         Tag="cpu"     Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="5"  x:Name="ColMem"     Content="Memory"      Tag="mem"     Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="6"  x:Name="ColDisk"    Content="Disk"        Tag="disk"    Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="7"  x:Name="ColThreads" Content="Threads"     Tag="threads" Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <Button Grid.Column="8"  x:Name="ColHandles" Content="Handles"     Tag="handles" Style="{StaticResource ColBtn}" Click="Sort_Click"/>
                    <TextBlock Grid.Column="9" Text="Arch" Foreground="#6C7086" FontSize="10" FontWeight="Bold" VerticalAlignment="Center"/>
                    <TextBlock Grid.Column="10" Text="Description" Foreground="#6C7086" FontSize="10" FontWeight="Bold" VerticalAlignment="Center" Margin="8,0,0,0"/>
                </Grid>

                <ListBox x:Name="LstProcs" Style="{StaticResource PlainList}">
                    <ListBox.ItemTemplate>
                        <DataTemplate>
                            <Border BorderBrush="#313244" BorderThickness="0,0,0,1" Padding="0,3">
                                <Grid>
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition Width="2*"/>
                                        <ColumnDefinition Width="56"/>
                                        <ColumnDefinition Width="100"/>
                                        <ColumnDefinition Width="110"/>
                                        <ColumnDefinition Width="64"/>
                                        <ColumnDefinition Width="84"/>
                                        <ColumnDefinition Width="76"/>
                                        <ColumnDefinition Width="62"/>
                                        <ColumnDefinition Width="62"/>
                                        <ColumnDefinition Width="48"/>
                                        <ColumnDefinition Width="3*"/>
                                        <ColumnDefinition Width="54"/>
                                    </Grid.ColumnDefinitions>
                                    <TextBlock Grid.Column="0" Text="{Binding Name}" ToolTip="{Binding PathTip}"
                                               Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="1" Text="{Binding Pid}" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="2" Text="{Binding Status}" FontSize="11" VerticalAlignment="Center">
                                        <TextBlock.Style>
                                            <Style TargetType="TextBlock">
                                                <Setter Property="Foreground" Value="#9399B2"/>
                                                <Style.Triggers>
                                                    <DataTrigger Binding="{Binding Status}" Value="Not responding">
                                                        <Setter Property="Foreground" Value="#F38BA8"/>
                                                    </DataTrigger>
                                                </Style.Triggers>
                                            </Style>
                                        </TextBlock.Style>
                                    </TextBlock>
                                    <TextBlock Grid.Column="3" Text="{Binding User}" Foreground="#9399B2" FontSize="11" TextTrimming="CharacterEllipsis" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="4" Text="{Binding CpuText}" Foreground="#CDD6F4" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="5" Text="{Binding MemText}" Foreground="#CDD6F4" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="6" Text="{Binding DiskText}" Foreground="#9399B2" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="7" Text="{Binding Threads}" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="8" Text="{Binding Handles}" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="9" Text="{Binding Arch}" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                                    <TextBlock Grid.Column="10" Text="{Binding Desc}" Foreground="#6C7086" FontSize="11" TextTrimming="CharacterEllipsis" VerticalAlignment="Center" Margin="8,0,0,0"/>
                                    <Button Grid.Column="11" Content="End" FontSize="10" Padding="8,2" HorizontalAlignment="Right"
                                            Style="{StaticResource ActionButton}" Tag="{Binding}" Click="EndTask_Click"
                                            IsEnabled="{Binding CanEnd}"/>
                                </Grid>
                            </Border>
                        </DataTemplate>
                    </ListBox.ItemTemplate>
                </ListBox>
            </DockPanel>
        </Border>
    </Grid>
</UserControl>
```
