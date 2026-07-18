---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\UpdatesPage.xaml
---

# PartnerTool\Pages\UpdatesPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.UpdatesPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:pt="clr-namespace:PartnerTool">

    <UserControl.Resources>
        <!-- Inline log scroller forwards the mouse wheel to the page when it can't scroll further -->
        <Style TargetType="ScrollViewer">
            <Setter Property="pt:ScrollChaining.Enabled" Value="True"/>
        </Style>
    </UserControl.Resources>

    <ScrollViewer VerticalScrollBarVisibility="Auto" Background="#1E1E2E">
        <StackPanel Margin="20,16,20,16">

            <TextBlock Text="Each source below is scanned automatically when this tab opens (can take a minute). Its status and its “Open” button live together — use Update All to run everything at once."
                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="2,0,2,12"/>

            <!-- WINDOWS UPDATE (launcher + pending scan, merged) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button DockPanel.Dock="Right" Content="Open" Style="{StaticResource ActionButton}"
                                VerticalAlignment="Top" Click="OpenWindowsUpdate_Click"/>
                        <StackPanel Margin="0,0,12,0">
                            <TextBlock Text="Windows Update" FontSize="14" FontWeight="SemiBold" Foreground="#CDD6F4"/>
                            <TextBlock Text="Latest Windows updates and security patches."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,3,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Rectangle Style="{StaticResource RowDivider}" Margin="0,10"/>
                    <TextBlock Text="PENDING UPDATES" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                    <ItemsControl x:Name="IcPending">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,3">
                                    <TextBlock DockPanel.Dock="Right" Foreground="#6C7086" FontSize="11" VerticalAlignment="Top"
                                               Text="{Binding SizeText, Mode=OneWay}"/>
                                    <TextBlock DockPanel.Dock="Left" Width="90" Foreground="#9399B2" FontSize="11" VerticalAlignment="Top"
                                               Text="{Binding Kb}"/>
                                    <TextBlock Text="{Binding Title}" Foreground="#CDD6F4" FontSize="11" TextWrapping="Wrap"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoPending" Foreground="#6C7086" FontSize="12" Text="Not scanned yet."/>
                </StackPanel>
            </Border>

            <!-- APP UPDATES (winget scan, merged) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <TextBlock Text="App Updates (winget)" FontSize="14" FontWeight="SemiBold" Foreground="#CDD6F4"/>
                    <TextBlock Text="Outdated apps detected by winget — upgraded by Update All below."
                               FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,3,0,8"/>
                    <ItemsControl x:Name="IcOutdated">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,3">
                                    <TextBlock DockPanel.Dock="Right" Foreground="#9399B2" FontSize="11" VerticalAlignment="Top">
                                        <Run Text="{Binding Current, Mode=OneWay}" Foreground="#6C7086"/>
                                        <Run Text=" → "/>
                                        <Run Text="{Binding Available, Mode=OneWay}" Foreground="#A6E3A1"/>
                                    </TextBlock>
                                    <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoOutdated" Foreground="#6C7086" FontSize="12" Text="Not scanned yet."/>
                </StackPanel>
            </Border>

            <!-- MANUFACTURER DRIVERS / BIOS (launcher + vendor scan, merged) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnMfr" DockPanel.Dock="Right" Content="Open" Style="{StaticResource ActionButton}"
                                VerticalAlignment="Top" IsEnabled="False" Click="OpenMfr_Click"/>
                        <StackPanel Margin="0,0,12,0">
                            <TextBlock x:Name="TxtMfrName" Text="Detecting manufacturer..."
                                       FontSize="14" FontWeight="SemiBold" Foreground="#CDD6F4"/>
                            <TextBlock x:Name="TxtMfrDesc" Text="" FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,3,0,0"/>
                            <TextBlock x:Name="TxtMfrStatus" Text="" FontSize="11" Margin="0,3,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Rectangle Style="{StaticResource RowDivider}" Margin="0,10"/>
                    <TextBlock Text="DRIVER / BIOS SCAN" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                    <ItemsControl x:Name="IcVendor">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,3">
                                    <TextBlock DockPanel.Dock="Right" Foreground="#9399B2" FontSize="11" VerticalAlignment="Top"
                                               Margin="10,0,0,0" Text="{Binding Detail}"/>
                                    <TextBlock Text="{Binding Title}" Foreground="#CDD6F4" FontSize="11" TextWrapping="Wrap"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoVendor" Foreground="#6C7086" FontSize="12" Text="Not scanned yet."/>
                </StackPanel>
            </Border>

            <!-- MICROSOFT STORE (launcher + pending scan, merged) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button DockPanel.Dock="Right" Content="Open" Style="{StaticResource ActionButton}"
                                VerticalAlignment="Top" Click="OpenStore_Click"/>
                        <StackPanel Margin="0,0,12,0">
                            <TextBlock Text="Microsoft Store" FontSize="14" FontWeight="SemiBold" Foreground="#CDD6F4"/>
                            <TextBlock Text="Store (UWP/MSIX) app updates — installed by Update All below."
                                       FontSize="11" Foreground="#6C7086" Margin="0,3,0,0"/>
                        </StackPanel>
                    </DockPanel>
                    <Rectangle Style="{StaticResource RowDivider}" Margin="0,10"/>
                    <TextBlock Text="PENDING STORE UPDATES" Foreground="#6C7086" FontSize="10" FontWeight="Bold" Margin="0,0,0,4"/>
                    <ItemsControl x:Name="IcStore">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding}" Foreground="#CDD6F4" FontSize="11" Margin="0,3" TextTrimming="CharacterEllipsis"/>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoStore" Foreground="#6C7086" FontSize="12" Text="Not scanned yet."/>
                </StackPanel>
            </Border>

            <!-- UPDATE ALL (log lives here, appears when it runs) -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnUpdateAll" DockPanel.Dock="Right" Content="Start All Updates"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Top"
                                Click="UpdateAll_Click"/>
                        <StackPanel Margin="0,0,16,0">
                            <TextBlock Text="UPDATE ALL" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Sequentially runs: Windows Defender signatures, Windows Update, manufacturer drivers (Dell/Lenovo/HP), all apps via winget, Microsoft Store apps, then launches the Microsoft Office updater (which opens its own progress window)."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        </StackPanel>
                    </DockPanel>

                    <StackPanel x:Name="PnlTasks" Visibility="Collapsed" Margin="0,16,0,0">
                        <ItemsControl x:Name="TaskList">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,5">
                                        <TextBlock DockPanel.Dock="Right" Text="{Binding StatusText}"
                                                   Foreground="{Binding StatusColor}" FontSize="11"
                                                   MinWidth="140" TextAlignment="Right" VerticalAlignment="Center"/>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"
                                                   VerticalAlignment="Center"/>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>

                        <!-- Live output log — only shown while/after Update All runs -->
                        <DockPanel Margin="0,14,0,8">
                            <CheckBox x:Name="ChkAutoScroll" DockPanel.Dock="Right"
                                      Content="Auto-scroll" IsChecked="True"
                                      Foreground="#6C7086" FontSize="10" VerticalAlignment="Center"/>
                            <TextBlock Text="LOG" Foreground="#6C7086" FontSize="10" FontWeight="Bold"
                                       VerticalAlignment="Center" Margin="0"/>
                        </DockPanel>
                        <Border Background="#11111B" CornerRadius="6" Padding="10,8">
                            <ScrollViewer x:Name="LogScroll" Height="180" VerticalScrollBarVisibility="Auto">
                                <TextBlock x:Name="TxtLog" Foreground="#6C7086" FontSize="10"
                                           FontFamily="Consolas" TextWrapping="Wrap"/>
                            </ScrollViewer>
                        </Border>
                    </StackPanel>
                </StackPanel>
            </Border>

            <!-- WINDOWS UPDATE HISTORY -->
            <Border Style="{StaticResource Card}">
                <StackPanel>
                    <DockPanel>
                        <Button x:Name="BtnMoreHistory" DockPanel.Dock="Right" Content="More…"
                                Style="{StaticResource ActionButton}" Padding="12,4" FontSize="11"
                                Click="MoreHistory_Click" Visibility="Collapsed"/>
                        <TextBlock Text="WINDOWS UPDATE HISTORY" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock x:Name="TxtLastPatched" Foreground="#CDD6F4" FontSize="12" Margin="0,2,0,6"/>
                    <ItemsControl x:Name="IcHistory">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,3">
                                    <TextBlock DockPanel.Dock="Left" Width="120" Foreground="#9399B2" FontSize="11" VerticalAlignment="Top"
                                               Text="{Binding Date, StringFormat={}{0:d MMM yyyy}, Mode=OneWay}"/>
                                    <TextBlock DockPanel.Dock="Right" Text="{Binding Result}" FontSize="11" Foreground="#6C7086" Margin="10,0,0,0" VerticalAlignment="Top"/>
                                    <TextBlock Text="{Binding Title}" Foreground="#CDD6F4" FontSize="11" TextWrapping="Wrap"/>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoHistory" Text="Loading update history…" Foreground="#6C7086" FontSize="12"/>
                </StackPanel>
            </Border>

        </StackPanel>
    </ScrollViewer>
</UserControl>
```
