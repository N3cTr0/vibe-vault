---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\App.xaml
---

# PartnerTool\App.xaml

```xml
<Application x:Class="PartnerTool.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <Application.Resources>

        <!-- Show/hide helper used across pages -->
        <BooleanToVisibilityConverter x:Key="BoolToVis"/>

        <!-- Virtualized, selection-free list for large tables (services, drivers, tasks…) -->
        <Style x:Key="PlainList" TargetType="ListBox">
            <Setter Property="Background" Value="Transparent"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled"/>
            <Setter Property="VirtualizingPanel.IsVirtualizing" Value="True"/>
            <Setter Property="VirtualizingPanel.VirtualizationMode" Value="Recycling"/>
            <Setter Property="ItemContainerStyle">
                <Setter.Value>
                    <Style TargetType="ListBoxItem">
                        <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
                        <Setter Property="Padding" Value="0"/>
                        <Setter Property="Template">
                            <Setter.Value>
                                <ControlTemplate TargetType="ListBoxItem">
                                    <!-- Right inset so row content/buttons don't crowd the scrollbar -->
                                    <ContentPresenter Margin="0,0,12,0"/>
                                </ControlTemplate>
                            </Setter.Value>
                        </Setter>
                    </Style>
                </Setter.Value>
            </Setter>
        </Style>

        <!-- ── Cards ── -->
        <Style x:Key="Card" TargetType="Border">
            <Setter Property="Background" Value="#313244"/>
            <Setter Property="CornerRadius" Value="8"/>
            <Setter Property="Padding" Value="18,14"/>
            <Setter Property="Margin" Value="0,0,0,12"/>
        </Style>
        <Style x:Key="CardTitle" TargetType="TextBlock">
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="Foreground" Value="#CBA6F7"/>
            <Setter Property="Margin" Value="0,0,0,8"/>
        </Style>

        <!-- ── Info rows ── -->
        <Style x:Key="RowLabel" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#6C7086"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="VerticalAlignment" Value="Top"/>
            <Setter Property="Padding" Value="0,5,0,5"/>
        </Style>
        <Style x:Key="RowValue" TargetType="TextBlock">
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Padding" Value="0,5,0,5"/>
            <Setter Property="TextWrapping" Value="Wrap"/>
        </Style>
        <Style x:Key="RowDivider" TargetType="Rectangle">
            <Setter Property="Height" Value="1"/>
            <Setter Property="Fill" Value="#45475A"/>
        </Style>

        <!-- ── Shared button template ── -->
        <ControlTemplate x:Key="FlatBtnTpl" TargetType="Button">
            <Border x:Name="bg" Background="{TemplateBinding Background}"
                    CornerRadius="6" Padding="{TemplateBinding Padding}">
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="bg" Property="Background" Value="#585B70"/>
                </Trigger>
                <Trigger Property="IsPressed" Value="True">
                    <Setter TargetName="bg" Property="Background" Value="#6C7086"/>
                </Trigger>
                <Trigger Property="IsEnabled" Value="False">
                    <Setter TargetName="bg" Property="Background" Value="#313244"/>
                    <Setter Property="Opacity" Value="0.4"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>

        <!-- ── Action button (small, inline) ── -->
        <Style x:Key="ActionButton" TargetType="Button">
            <Setter Property="Background" Value="#45475A"/>
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Padding" Value="18,8"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template" Value="{StaticResource FlatBtnTpl}"/>
        </Style>

        <!-- ── Tool button (larger, grid tiles) ── -->
        <Style x:Key="ToolButton" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="Height" Value="50"/>
            <Setter Property="Padding" Value="8,0"/>
            <Setter Property="Margin" Value="0,0,8,8"/>
            <Setter Property="FontSize" Value="12"/>
        </Style>

        <!-- ── Progress bar (dark theme) ── -->
        <ControlTemplate x:Key="ProgressTpl" TargetType="ProgressBar">
            <Grid>
                <Border x:Name="PART_Track" CornerRadius="4" Background="{TemplateBinding Background}"/>
                <Border x:Name="PART_Indicator" CornerRadius="4"
                        Background="{TemplateBinding Foreground}"
                        HorizontalAlignment="Left"/>
            </Grid>
        </ControlTemplate>
        <Style x:Key="UsageBar" TargetType="ProgressBar">
            <Setter Property="Height" Value="8"/>
            <Setter Property="Maximum" Value="100"/>
            <Setter Property="Background" Value="#45475A"/>
            <Setter Property="Foreground" Value="#89B4FA"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Template" Value="{StaticResource ProgressTpl}"/>
        </Style>

    </Application.Resources>
</Application>
```
