---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\AboutWindow.xaml
---

# PartnerTool\AboutWindow.xaml

```xml
<Window x:Class="PartnerTool.AboutWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="About Partner Tool"
        Width="470" SizeToContent="Height"
        ResizeMode="NoResize" WindowStartupLocation="CenterOwner"
        ShowInTaskbar="False" Background="#1E1E2E"
        Icon="Resources/logo.ico">

    <Border Padding="22,20">
        <StackPanel>
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>

                <Image Grid.Column="0" Source="Resources/logo.png" Width="92" Height="92"
                       VerticalAlignment="Top" Margin="0,2,20,0"
                       RenderOptions.BitmapScalingMode="HighQuality"/>

                <StackPanel Grid.Column="1">
                    <TextBlock Text="Progressive Computing" FontSize="16" FontWeight="SemiBold"
                               Foreground="#CDD6F4"/>

                    <TextBlock Margin="0,8,0,0" FontSize="12">
                        <Hyperlink NavigateUri="https://www.progressivecomputing.com"
                                   RequestNavigate="Link_RequestNavigate" Foreground="#89B4FA">
                            www.progressivecomputing.com
                        </Hyperlink>
                    </TextBlock>

                    <TextBlock Margin="0,8,0,0" FontSize="12" ToolTip="Click a number to copy it">
                        <Hyperlink Tag="914-375-3009" Click="Phone_Click"
                                   Foreground="#CDD6F4" TextDecorations="{x:Null}">914-375-3009</Hyperlink>
                        <Run Text="  |  " Foreground="#6C7086"/>
                        <Hyperlink Tag="212-681-1212" Click="Phone_Click"
                                   Foreground="#CDD6F4" TextDecorations="{x:Null}">212-681-1212</Hyperlink>
                        <Run Text="  |  " Foreground="#6C7086"/>
                        <Hyperlink Tag="860-691-0044" Click="Phone_Click"
                                   Foreground="#CDD6F4" TextDecorations="{x:Null}">860-691-0044</Hyperlink>
                    </TextBlock>

                    <TextBlock Text="51 Smart Ave, Yonkers, NY 10704"
                               Foreground="#A6ADC8" FontSize="12" Margin="0,8,0,0"/>
                    <TextBlock Text="34 Black Point Road, Niantic, CT 06357"
                               Foreground="#A6ADC8" FontSize="12" Margin="0,2,0,0"/>

                    <StackPanel Orientation="Horizontal" Margin="0,10,0,0">
                        <TextBlock FontSize="12">
                            <Hyperlink NavigateUri="http://www.linkedin.com/company/129107"
                                       RequestNavigate="Link_RequestNavigate" Foreground="#89B4FA">
                                LinkedIn
                            </Hyperlink>
                        </TextBlock>
                        <TextBlock Text="   •   " Foreground="#6C7086" FontSize="12"/>
                        <TextBlock FontSize="12">
                            <Hyperlink NavigateUri="https://www.facebook.com/progressivecomputing"
                                       RequestNavigate="Link_RequestNavigate" Foreground="#89B4FA">
                                Facebook
                            </Hyperlink>
                        </TextBlock>
                    </StackPanel>

                    <TextBlock Margin="0,12,0,0" FontSize="12" TextWrapping="Wrap" Foreground="#A6ADC8">
                        <Run Text="Suggestions / feedback: "/>
                        <Hyperlink NavigateUri="mailto:graemel@progressivecomputing.com?subject=Feedback%20on%20Partner%20Support%20Tool"
                                   RequestNavigate="Link_RequestNavigate" Foreground="#89B4FA">graemel@progressivecomputing.com</Hyperlink>
                    </TextBlock>
                </StackPanel>
            </Grid>

            <Rectangle Height="1" Fill="#313244" Margin="0,18,0,12"/>

            <DockPanel>
                <TextBlock x:Name="TxtVersion" DockPanel.Dock="Left"
                           Foreground="#6C7086" FontSize="11" VerticalAlignment="Center"/>
                <Button Content="OK" DockPanel.Dock="Right" HorizontalAlignment="Right"
                        Style="{StaticResource ActionButton}" MinWidth="80"
                        IsDefault="True" IsCancel="True" Click="Ok_Click"/>
            </DockPanel>

            <!-- "Copied!" toast shown at the cursor (no layout footprint) -->
            <Popup x:Name="CopiedPopup" Placement="Mouse" AllowsTransparency="True" StaysOpen="True">
                <Border Background="#313244" CornerRadius="4" Padding="9,5" Margin="6">
                    <Border.Effect>
                        <DropShadowEffect BlurRadius="8" ShadowDepth="1" Opacity="0.5" Color="Black"/>
                    </Border.Effect>
                    <TextBlock Text="Copied!" Foreground="#A6E3A1" FontSize="11" FontWeight="SemiBold"/>
                </Border>
            </Popup>
        </StackPanel>
    </Border>
</Window>
```
