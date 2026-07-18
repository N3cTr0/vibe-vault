---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\SecurityPage.xaml
---

# PartnerTool\Pages\SecurityPage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.SecurityPage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Grid Background="#1E1E2E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <DockPanel Grid.Row="0" Margin="20,12,20,4">
            <TextBlock Text="SECURITY POSTURE" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
        </DockPanel>

        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
            <StackPanel Margin="20,4,20,16">

                <!-- HARDENING SCORECARD -->
                <Border Style="{StaticResource Card}">
                    <StackPanel>
                        <TextBlock Text="HARDENING SCORECARD" Style="{StaticResource CardTitle}"/>
                        <ItemsControl x:Name="IcAudit">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <DockPanel Margin="0,5">
                                        <Ellipse DockPanel.Dock="Left" Width="9" Height="9" VerticalAlignment="Center" Margin="0,0,10,0">
                                            <Ellipse.Style>
                                                <Style TargetType="Ellipse">
                                                    <Setter Property="Fill" Value="#6C7086"/>
                                                    <Style.Triggers>
                                                        <DataTrigger Binding="{Binding Level}" Value="Good"><Setter Property="Fill" Value="#A6E3A1"/></DataTrigger>
                                                        <DataTrigger Binding="{Binding Level}" Value="Warn"><Setter Property="Fill" Value="#F9E2AF"/></DataTrigger>
                                                        <DataTrigger Binding="{Binding Level}" Value="Bad"><Setter Property="Fill" Value="#F38BA8"/></DataTrigger>
                                                        <DataTrigger Binding="{Binding Level}" Value="Info"><Setter Property="Fill" Value="#89B4FA"/></DataTrigger>
                                                    </Style.Triggers>
                                                </Style>
                                            </Ellipse.Style>
                                        </Ellipse>
                                        <!-- Plain value (no Windows setting to jump to) -->
                                        <TextBlock DockPanel.Dock="Right" Text="{Binding Detail}" Foreground="#9399B2"
                                                   FontSize="12" Margin="12,0,0,0" TextAlignment="Right">
                                            <TextBlock.Style>
                                                <Style TargetType="TextBlock">
                                                    <Setter Property="Visibility" Value="Collapsed"/>
                                                    <Style.Triggers>
                                                        <DataTrigger Binding="{Binding Fix}" Value="{x:Null}">
                                                            <Setter Property="Visibility" Value="Visible"/>
                                                        </DataTrigger>
                                                    </Style.Triggers>
                                                </Style>
                                            </TextBlock.Style>
                                        </TextBlock>
                                        <!-- Clickable value → opens the Windows setting to change it -->
                                        <TextBlock DockPanel.Dock="Right" FontSize="12" Margin="12,0,0,0" TextAlignment="Right">
                                            <TextBlock.Style>
                                                <Style TargetType="TextBlock">
                                                    <Setter Property="Visibility" Value="Visible"/>
                                                    <Style.Triggers>
                                                        <DataTrigger Binding="{Binding Fix}" Value="{x:Null}">
                                                            <Setter Property="Visibility" Value="Collapsed"/>
                                                        </DataTrigger>
                                                    </Style.Triggers>
                                                </Style>
                                            </TextBlock.Style>
                                            <Hyperlink Click="Fix_Click" Foreground="#89B4FA"
                                                       ToolTip="{Binding Fix.Tooltip}"><Run Text="{Binding Detail}"/></Hyperlink>
                                        </TextBlock>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"/>
                                    </DockPanel>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </Border>

                <!-- BITLOCKER RECOVERY KEY -->
                <Border Style="{StaticResource Card}">
                    <DockPanel>
                        <Button x:Name="BtnBitLocker" DockPanel.Dock="Right" Content="Show recovery key"
                                Style="{StaticResource ActionButton}" VerticalAlignment="Center"
                                Click="BitLocker_Click"/>
                        <StackPanel>
                            <TextBlock Text="BITLOCKER RECOVERY KEY" Style="{StaticResource CardTitle}"/>
                            <TextBlock Text="Reveal the 48-digit BitLocker recovery key(s) for this machine's drives — for unlocking a drive after a TPM/hardware change. Keys are read on demand and only shown when you click."
                                       FontSize="11" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,0,12,0"/>
                        </StackPanel>
                    </DockPanel>
                </Border>

                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="12"/>
                        <ColumnDefinition Width="*"/>
                    </Grid.ColumnDefinitions>

                    <!-- DEFENDER -->
                    <Border Grid.ColumnSpan="3" Style="{StaticResource Card}" VerticalAlignment="Top">
                        <StackPanel>
                            <TextBlock Text="MICROSOFT DEFENDER" Style="{StaticResource CardTitle}"/>
                            <TextBlock x:Name="TxtNoDefender" Foreground="#6C7086" FontSize="12"
                                       Text="Defender is not the active antivirus (third-party AV installed)." Visibility="Collapsed"/>
                            <Grid x:Name="DefenderGrid">
                                <Grid.ColumnDefinitions><ColumnDefinition Width="150"/><ColumnDefinition Width="*"/></Grid.ColumnDefinitions>
                                <Grid.RowDefinitions>
                                    <RowDefinition/><RowDefinition/><RowDefinition/><RowDefinition/>
                                    <RowDefinition/><RowDefinition/><RowDefinition/>
                                </Grid.RowDefinitions>
                                <TextBlock Grid.Row="0" Grid.Column="0" Text="Real-time protection" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="0" Grid.Column="1" x:Name="TxtRtp" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="1" Grid.Column="0" Text="Tamper protection" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="1" Grid.Column="1" x:Name="TxtTamper" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="2" Grid.Column="0" Text="Signature version" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="2" Grid.Column="1" x:Name="TxtSig" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="3" Grid.Column="0" Text="Signatures updated" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="3" Grid.Column="1" x:Name="TxtSigDate" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="4" Grid.Column="0" Text="Last quick scan" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="4" Grid.Column="1" x:Name="TxtQuick" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="5" Grid.Column="0" Text="Last full scan" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="5" Grid.Column="1" x:Name="TxtFull" Style="{StaticResource RowValue}"/>
                                <TextBlock Grid.Row="6" Grid.Column="0" Text="Threats" Style="{StaticResource RowLabel}"/>
                                <TextBlock Grid.Row="6" Grid.Column="1" x:Name="TxtThreats" Style="{StaticResource RowValue}"/>
                            </Grid>
                        </StackPanel>
                    </Border>
                </Grid>

            </StackPanel>
        </ScrollViewer>
    </Grid>
</UserControl>
```
