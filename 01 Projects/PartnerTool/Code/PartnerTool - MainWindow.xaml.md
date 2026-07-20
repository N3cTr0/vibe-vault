---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\MainWindow.xaml
---

# PartnerTool\MainWindow.xaml

```xml
<Window x:Class="PartnerTool.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Partner Tool" Height="900" Width="1140"
        MinHeight="560" MinWidth="900"
        ResizeMode="CanResize"
        WindowStartupLocation="Manual"
        Background="#1E1E2E"
        Icon="Resources/logo.ico">

    <Window.Resources>
        <ControlTemplate x:Key="NavBtnTpl" TargetType="Button">
            <Border x:Name="bg" Background="{TemplateBinding Background}"
                    CornerRadius="6" Padding="{TemplateBinding Padding}">
                <ContentPresenter HorizontalAlignment="Left" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="bg" Property="Background" Value="#2A2B3C"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>

        <Style x:Key="NavBtn" TargetType="Button">
            <Setter Property="Background" Value="Transparent"/>
            <Setter Property="Foreground" Value="#6C7086"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Padding" Value="14,9"/>
            <Setter Property="HorizontalContentAlignment" Value="Left"/>
            <Setter Property="FontSize" Value="13"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template" Value="{StaticResource NavBtnTpl}"/>
            <Setter Property="Margin" Value="0,2"/>
        </Style>
        <Style x:Key="NavBtnActive" TargetType="Button" BasedOn="{StaticResource NavBtn}">
            <Setter Property="Background" Value="#313244"/>
            <Setter Property="Foreground" Value="#CDD6F4"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
        </Style>

        <ControlTemplate x:Key="ExportBtnTpl" TargetType="Button">
            <Border x:Name="bg" Background="{TemplateBinding Background}"
                    CornerRadius="6" Padding="{TemplateBinding Padding}">
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="bg" Property="Background" Value="#1E3A5F"/>
                </Trigger>
                <Trigger Property="IsPressed" Value="True">
                    <Setter TargetName="bg" Property="Background" Value="#164070"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>
        <Style x:Key="ExportBtn" TargetType="Button">
            <Setter Property="Background" Value="#1C3654"/>
            <Setter Property="Foreground" Value="#89B4FA"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Padding" Value="14,9"/>
            <Setter Property="FontSize" Value="12"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template" Value="{StaticResource ExportBtnTpl}"/>
        </Style>

        <!-- Tech-gate lock pill in the header. Background/foreground are set in code as the state flips. -->
        <ControlTemplate x:Key="GateBtnTpl" TargetType="Button">
            <Border x:Name="bg" Background="{TemplateBinding Background}" CornerRadius="12"
                    Padding="{TemplateBinding Padding}" BorderThickness="1" BorderBrush="#313244">
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="bg" Property="BorderBrush" Value="#585B70"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>
    </Window.Resources>

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Header -->
        <Border Grid.Row="0" Background="#181825" Padding="20,10">
            <DockPanel>
                <Image DockPanel.Dock="Right" Source="Resources/logo.png" Height="48" Width="48"
                       RenderOptions.BitmapScalingMode="HighQuality" Margin="12,0,0,0"/>

                <!-- Tech-gate state: padlock + countdown while unlocked; click to lock out by hand. -->
                <Button DockPanel.Dock="Right" x:Name="BtnGate" Style="{StaticResource NavBtn}"
                        Template="{StaticResource GateBtnTpl}" Padding="10,5" Margin="0,0,4,0"
                        VerticalAlignment="Center" Click="Gate_Click">
                    <StackPanel Orientation="Horizontal">
                        <TextBlock x:Name="TxtGateIcon" Text="&#xE72E;" FontFamily="Segoe MDL2 Assets"
                                   FontSize="13" VerticalAlignment="Center"/>
                        <TextBlock x:Name="TxtGateText" Text="Locked" FontSize="11" Margin="6,0,0,0"
                                   VerticalAlignment="Center"/>
                    </StackPanel>
                </Button>

                <TextBlock Text="Partner Support Tool" FontSize="15" FontWeight="SemiBold"
                           Foreground="#CDD6F4" VerticalAlignment="Center"/>
            </DockPanel>
        </Border>

        <!-- Body -->
        <Grid Grid.Row="1">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="165"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>

            <!-- Sidebar -->
            <Border Grid.Column="0" Background="#181825">
                <DockPanel>
                    <!-- Health Check + Settings + About pinned to bottom (Health above Settings for now) -->
                    <StackPanel DockPanel.Dock="Bottom" Margin="8,8,8,12">
                        <Button x:Name="BtnNavHealth" Content="Health Check" Style="{StaticResource NavBtn}"
                                HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"
                                Margin="0,0,0,6" Click="NavBtn_Click" Tag="Health"/>
                        <Button Content="Settings" Style="{StaticResource NavBtn}"
                                AutomationProperties.AutomationId="BtnNavSettings"
                                HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"
                                Margin="0,0,0,0" Click="Settings_Click"/>
                        <Button Content="About" Style="{StaticResource NavBtn}"
                                AutomationProperties.AutomationId="BtnNavAbout"
                                HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"
                                Margin="0,6,0,0" Click="About_Click"/>
                    </StackPanel>
                    <!-- Nav buttons -->
                    <ScrollViewer VerticalScrollBarVisibility="Auto" HorizontalScrollBarVisibility="Disabled">
                    <StackPanel Margin="8,14,8,0">
                        <!-- System Info first (Processes rides with it); everything below alphabetical -->
                        <Button x:Name="BtnNavSysInfo"  Content="System Info"  Style="{StaticResource NavBtnActive}" Click="NavBtn_Click" Tag="SysInfo"/>
                        <Button x:Name="BtnNavProcesses" Content="Processes"   Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Processes"/>
                        <Button x:Name="BtnNavDiag"     Content="Diagnostics"  Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Diag"/>
                        <Button x:Name="BtnNavDiskUsage" Content="Disk Usage"  Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="DiskUsage"/>
                        <Button x:Name="BtnNavManage"   Content="Manage"       Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Manage"/>
                        <Button x:Name="BtnNavNetwork"  Content="Network"      Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Network"/>
                        <Button x:Name="BtnNavRepair"   Content="Repair"       Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Repair"/>
                        <Button x:Name="BtnNavSecurity" Content="Security"     Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Security"/>
                        <Button x:Name="BtnNavSoftware" Content="Software"     Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Software"/>
                        <Button x:Name="BtnNavShortcuts" Content="System Shortcuts" Style="{StaticResource NavBtn}"  Click="NavBtn_Click" Tag="Shortcuts"/>
                        <Button x:Name="BtnNavUpdates"  Content="Updates"      Style="{StaticResource NavBtn}"       Click="NavBtn_Click" Tag="Updates"/>
                    </StackPanel>
                    </ScrollViewer>
                </DockPanel>
            </Border>

            <!-- Sidebar/content divider -->
            <Rectangle Grid.Column="0" Width="1" HorizontalAlignment="Right" Fill="#313244"/>

            <!-- Page host -->
            <ContentControl Grid.Column="1" x:Name="PageHost"
                            HorizontalContentAlignment="Stretch"
                            VerticalContentAlignment="Stretch"/>
        </Grid>

        <!-- Loading splash — covers everything until the first snapshot is ready -->
        <Border x:Name="LoadingOverlay" Grid.Row="0" Grid.RowSpan="2" Background="#1E1E2E" Panel.ZIndex="100">
            <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
                <Image Source="Resources/logo.png" Width="72" Height="72"
                       RenderOptions.BitmapScalingMode="HighQuality" HorizontalAlignment="Center"/>
                <TextBlock x:Name="TxtLoading" Text="Analyzing System, Please Wait…" Foreground="#CDD6F4"
                           FontSize="17" FontWeight="SemiBold" Margin="0,20,0,0" HorizontalAlignment="Center"/>
                <TextBlock Text="Gathering hardware, performance, security and diagnostics"
                           Foreground="#6C7086" FontSize="12" Margin="0,8,0,0" HorizontalAlignment="Center"/>
                <ProgressBar IsIndeterminate="True" Width="260" Height="4" Margin="0,20,0,0"
                             Background="#313244" Foreground="#89B4FA" BorderThickness="0"/>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```
