---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\HealthCheckPage.xaml
---

# PartnerTool\Pages\HealthCheckPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.HealthCheckPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:pt="clr-namespace:PartnerTool">

    <UserControl.Resources>
        <Style TargetType="ScrollViewer">
            <Setter Property="pt:ScrollChaining.Enabled" Value="True"/>
        </Style>

        <!-- One finding row (shared by both columns) -->
        <DataTemplate x:Key="FindingTemplate">
            <Border Background="#242536" CornerRadius="8" Padding="12,10" Margin="0,0,0,8">
                <DockPanel>
                    <!-- Right: fix checkbox (fixable) or Open button (advisory) -->
                    <CheckBox DockPanel.Dock="Right" VerticalAlignment="Center" Margin="10,0,0,0"
                              Content="{Binding FixLabel}" IsChecked="{Binding Selected, Mode=TwoWay}"
                              Visibility="{Binding CheckVis}" Foreground="#CDD6F4" FontSize="11"/>
                    <Button DockPanel.Dock="Right" VerticalAlignment="Center" Margin="10,0,0,0"
                            Content="Open →" Tag="{Binding NavTag}" Click="Open_Click"
                            Visibility="{Binding NavVis}" Style="{StaticResource ActionButton}"
                            Padding="12,4" FontSize="11"/>

                    <!-- Severity dot -->
                    <Ellipse DockPanel.Dock="Left" Width="10" Height="10" VerticalAlignment="Top"
                             Margin="0,4,12,0">
                        <Ellipse.Style>
                            <Style TargetType="Ellipse">
                                <Setter Property="Fill" Value="#A6E3A1"/>
                                <Style.Triggers>
                                    <DataTrigger Binding="{Binding Severity}" Value="Warn">
                                        <Setter Property="Fill" Value="#F9E2AF"/>
                                    </DataTrigger>
                                    <DataTrigger Binding="{Binding Severity}" Value="Bad">
                                        <Setter Property="Fill" Value="#F38BA8"/>
                                    </DataTrigger>
                                </Style.Triggers>
                            </Style>
                        </Ellipse.Style>
                    </Ellipse>

                    <StackPanel>
                        <TextBlock Text="{Binding Title}" Foreground="#CDD6F4" FontWeight="SemiBold" FontSize="12"/>
                        <TextBlock Text="{Binding Detail}" Foreground="#9399B2" FontSize="11"
                                   TextWrapping="Wrap" Margin="0,2,0,0"/>
                    </StackPanel>
                </DockPanel>
            </Border>
        </DataTemplate>

        <!-- A category group: header + its findings. The per-row Category prefix is gone —
             the group header carries it now. -->
        <DataTemplate x:Key="GroupTemplate">
            <StackPanel Margin="0,0,0,10">
                <DockPanel Margin="2,0,2,6">
                    <TextBlock DockPanel.Dock="Right" Text="{Binding Summary}" Foreground="#6C7086"
                               FontSize="11" VerticalAlignment="Center"/>
                    <TextBlock Text="{Binding Name}" Foreground="#B4BEFE" FontSize="11"
                               FontWeight="Bold" VerticalAlignment="Center"/>
                </DockPanel>
                <ItemsControl ItemsSource="{Binding Items}" ItemTemplate="{StaticResource FindingTemplate}"/>
            </StackPanel>
        </DataTemplate>
    </UserControl.Resources>

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <StackPanel Margin="20,16,20,16">

            <TextBlock Text="HEALTH CHECK" Style="{StaticResource CardTitle}"/>
            <TextBlock Text="One scan of everything the tool already checks — junk, disk, updates, security, stability — with a score and one-click fixes for the safe items. No registry cleaning or fake tune-ups."
                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,12"/>

            <!-- Score ring (click to scan / rescan) -->
            <Border x:Name="Ring" Width="200" Height="200" CornerRadius="100" Cursor="Hand"
                    HorizontalAlignment="Center" Margin="0,4,0,4" MouseLeftButtonUp="Ring_Click">
                <Grid>
                    <Ellipse x:Name="EllRing" Stroke="#45475A" StrokeThickness="10" Fill="#181825"/>
                    <!-- A single bright arc that spins while scanning (rotation animated in code) -->
                    <Ellipse x:Name="SpinArc" Stroke="#89B4FA" StrokeThickness="10" StrokeDashCap="Round"
                             StrokeDashArray="18 200" Visibility="Collapsed" RenderTransformOrigin="0.5,0.5">
                        <Ellipse.RenderTransform><RotateTransform x:Name="SpinRot" Angle="0"/></Ellipse.RenderTransform>
                    </Ellipse>
                    <StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
                        <TextBlock x:Name="TxtScore" Text="SCAN" FontSize="40" FontWeight="Bold"
                                   Foreground="#89B4FA" HorizontalAlignment="Center"/>
                        <TextBlock x:Name="TxtScoreSub" Text="click to run" FontSize="12"
                                   Foreground="#6C7086" HorizontalAlignment="Center" Margin="0,-2,0,0"/>
                    </StackPanel>
                </Grid>
            </Border>
            <TextBlock x:Name="TxtStatus" Text="Click the circle to scan this machine."
                       Foreground="#9399B2" FontSize="12" HorizontalAlignment="Center" Margin="0,0,0,2"/>
            <ProgressBar x:Name="ProgBar" Height="6" Width="280" Minimum="0" Maximum="100"
                         Background="#313244" Foreground="#89B4FA" BorderThickness="0"
                         HorizontalAlignment="Center" Visibility="Collapsed" Margin="0,4,0,8"/>

            <!-- FINDINGS -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnFixSelected" DockPanel.Dock="Right" Content="Fix Selected"
                                Style="{StaticResource ActionButton}" IsEnabled="False" Click="FixSelected_Click"/>
                        <TextBlock Text="FINDINGS" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>

                    <TextBlock x:Name="TxtNoFindings" Text="Run a scan to see results."
                               Foreground="#6C7086" FontSize="12" Margin="0,6,0,0"/>

                    <!-- Findings, grouped by category, balanced across two columns -->
                    <Grid Margin="0,6,0,0">
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="*"/>
                        </Grid.ColumnDefinitions>
                        <ItemsControl x:Name="IcGroupsL" Grid.Column="0" Margin="0,0,6,0"
                                      ItemTemplate="{StaticResource GroupTemplate}"/>
                        <ItemsControl x:Name="IcGroupsR" Grid.Column="1" Margin="6,0,0,0"
                                      ItemTemplate="{StaticResource GroupTemplate}"/>
                    </Grid>

                    <!-- Fix log — shown while/after Fix Selected runs -->
                    <Border x:Name="FixLogPanel" Background="#11111B" CornerRadius="6" Padding="10,8"
                            Margin="0,4,0,0" Visibility="Collapsed">
                        <ScrollViewer x:Name="FixLogScroll" Height="120" VerticalScrollBarVisibility="Auto">
                            <TextBlock x:Name="TxtFixLog" Foreground="#9399B2" FontSize="10"
                                       FontFamily="Consolas" TextWrapping="Wrap"/>
                        </ScrollViewer>
                    </Border>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
