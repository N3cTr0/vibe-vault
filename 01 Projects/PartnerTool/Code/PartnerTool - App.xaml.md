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

        <!-- Decile gridlines for the live graphs: 10 equal bands, so each band is 10% of that
             graph's scale (100% for CPU/Memory/Disk; 10% of the auto-scaled max for Network).
             Drop `<Control Template="{StaticResource GraphGrid}"/>` behind the Polyline - it's
             hit-test-transparent so it can sit inside clickable tiles. -->
        <ControlTemplate x:Key="GraphGrid" TargetType="Control">
            <UniformGrid Rows="10" Columns="1" IsHitTestVisible="False">
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <!-- 50% line slightly brighter as a reference midpoint -->
                <Border BorderBrush="#303348" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
                <Border BorderBrush="#252739" BorderThickness="0,1,0,0"/>
            </UniformGrid>
        </ControlTemplate>

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

        <!-- Divider for ItemsControl rows. Sits at the TOP of every row and collapses on the
             first one (PreviousData is null only for item 0), so a list gets lines *between*
             its rows and none trailing off the end.
             Set Margin at the use site, and SPLIT IT top/bottom - the row's own top margin lands
             ABOVE this line, so a bottom-only margin leaves the text hugging the line above it.
             For a row with Margin="0,N", use Margin="0,N" here too: N+N above, N+N below. -->
        <Style x:Key="ItemDivider" TargetType="Rectangle" BasedOn="{StaticResource RowDivider}">
            <Style.Triggers>
                <DataTrigger Binding="{Binding RelativeSource={RelativeSource PreviousData}}" Value="{x:Null}">
                    <Setter Property="Visibility" Value="Collapsed"/>
                </DataTrigger>
            </Style.Triggers>
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

        <!-- ── Drop-down (dark theme) ── -->
        <Style x:Key="ComboItem" TargetType="ComboBoxItem">
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Padding" Value="10,6"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ComboBoxItem">
                        <Border x:Name="bg" Background="Transparent" CornerRadius="4"
                                Padding="{TemplateBinding Padding}">
                            <ContentPresenter VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsHighlighted" Value="True">
                                <Setter TargetName="bg" Property="Background" Value="#45475A"/>
                            </Trigger>
                            <Trigger Property="IsSelected" Value="True">
                                <Setter TargetName="bg" Property="Background" Value="#585B70"/>
                                <Setter Property="Foreground" Value="#CBA6F7"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <Style x:Key="DarkCombo" TargetType="ComboBox">
            <Setter Property="Background" Value="#45475A"/>
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Height" Value="30"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="ItemContainerStyle" Value="{StaticResource ComboItem}"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ComboBox">
                        <Border x:Name="bg" Background="{TemplateBinding Background}" CornerRadius="6">
                            <Grid>
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="*"/>
                                    <ColumnDefinition Width="Auto"/>
                                </Grid.ColumnDefinitions>
                                <ContentPresenter Grid.Column="0" Margin="10,0,0,0"
                                                  VerticalAlignment="Center" IsHitTestVisible="False"
                                                  Content="{TemplateBinding SelectionBoxItem}"
                                                  ContentTemplate="{TemplateBinding SelectionBoxItemTemplate}"
                                                  ContentTemplateSelector="{TemplateBinding ItemTemplateSelector}"
                                                  ContentStringFormat="{TemplateBinding SelectionBoxItemStringFormat}"
                                                  TextElement.Foreground="{TemplateBinding Foreground}"/>
                                <Path x:Name="arrow" Grid.Column="1" Margin="8,1,11,0" VerticalAlignment="Center"
                                      Data="M0,0 L4,4 L8,0" Stroke="#CDD6F4" StrokeThickness="1.4"
                                      StrokeStartLineCap="Round" StrokeEndLineCap="Round"
                                      IsHitTestVisible="False"/>
                                <!-- Transparent hit area over the whole box so a click anywhere opens it -->
                                <ToggleButton Grid.ColumnSpan="2" Focusable="False" ClickMode="Press"
                                              IsChecked="{Binding IsDropDownOpen, Mode=TwoWay,
                                                          RelativeSource={RelativeSource TemplatedParent}}">
                                    <ToggleButton.Template>
                                        <ControlTemplate TargetType="ToggleButton">
                                            <Border Background="Transparent"/>
                                        </ControlTemplate>
                                    </ToggleButton.Template>
                                </ToggleButton>
                                <Popup x:Name="PART_Popup" Placement="Bottom" Focusable="False"
                                       AllowsTransparency="True" PopupAnimation="Slide"
                                       IsOpen="{TemplateBinding IsDropDownOpen}">
                                    <Border Background="#1E1E2E" BorderBrush="#585B70" BorderThickness="1"
                                            CornerRadius="6" Margin="0,4,0,0" Padding="4"
                                            MinWidth="{Binding ActualWidth, RelativeSource={RelativeSource TemplatedParent}}">
                                        <ScrollViewer MaxHeight="{TemplateBinding MaxDropDownHeight}">
                                            <ItemsPresenter/>
                                        </ScrollViewer>
                                    </Border>
                                </Popup>
                            </Grid>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="bg" Property="Background" Value="#585B70"/>
                            </Trigger>
                            <Trigger Property="IsDropDownOpen" Value="True">
                                <Setter TargetName="bg" Property="Background" Value="#585B70"/>
                                <Setter TargetName="arrow" Property="Stroke" Value="#CBA6F7"/>
                            </Trigger>
                            <Trigger Property="IsEnabled" Value="False">
                                <Setter TargetName="bg" Property="Background" Value="#313244"/>
                                <Setter Property="Opacity" Value="0.4"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
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
