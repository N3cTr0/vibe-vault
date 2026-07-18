---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\ManagePage.xaml
---

# PartnerTool\Pages\ManagePage.xaml

```xml
<UserControl x:Class="PartnerTool.Pages.ManagePage"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <UserControl.Resources>
        <Style x:Key="SubNav" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="Margin" Value="0,0,6,0"/>
            <Setter Property="Padding" Value="12,6"/>
        </Style>

        <!-- Clickable sort headers for the services list -->
        <Style x:Key="SortBtn" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="Margin" Value="0,0,6,0"/>
            <Setter Property="Padding" Value="10,3"/>
            <Setter Property="FontSize" Value="11"/>
        </Style>

        <!-- Service action buttons follow state: Stop/Restart only when running, Start only when stopped. -->
        <Style x:Key="SvcShowWhenRunning" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Setter Property="Visibility" Value="Collapsed"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding Running}" Value="True">
                    <Setter Property="Visibility" Value="Visible"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
        <Style x:Key="SvcShowWhenStopped" TargetType="Button" BasedOn="{StaticResource ActionButton}">
            <Style.Triggers>
                <DataTrigger Binding="{Binding Running}" Value="True">
                    <Setter Property="Visibility" Value="Collapsed"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </UserControl.Resources>

    <Grid Background="#1E1E2E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Sub-navigation -->
        <WrapPanel Grid.Row="0" Margin="20,14,20,6">
            <!-- Services (most used) pinned first; the rest alphabetical -->
            <Button Content="Services"              Style="{StaticResource SubNav}" Tag="Services" Click="Sub_Click"/>
            <Button Content="Drivers"               Style="{StaticResource SubNav}" Tag="Drivers"  Click="Sub_Click"/>
            <Button Content="Environment Variables" Style="{StaticResource SubNav}" Tag="Env"      Click="Sub_Click"/>
            <Button Content="Features"              Style="{StaticResource SubNav}" Tag="Features" Click="Sub_Click"/>
            <Button Content="Hosts File"            Style="{StaticResource SubNav}" Tag="Hosts"    Click="Sub_Click"/>
            <Button Content="Scheduled Tasks"       Style="{StaticResource SubNav}" Tag="Tasks"    Click="Sub_Click"/>
            <Button Content="User Profiles"         Style="{StaticResource SubNav}" Tag="Profiles" Click="Sub_Click"/>
        </WrapPanel>

        <Grid Grid.Row="1" Margin="20,4,20,16">

            <!-- SERVICES -->
            <Border x:Name="SecServices" Style="{StaticResource Card}">
                <DockPanel>
                    <Border DockPanel.Dock="Top" Background="#45475A" CornerRadius="6" Padding="10,0" Margin="0,0,0,8">
                        <TextBox x:Name="TxtSvcSearch" Background="Transparent" Foreground="#CDD6F4" FontSize="12"
                                 BorderThickness="0" Height="30" VerticalContentAlignment="Center" TextChanged="SvcSearch_TextChanged"/>
                    </Border>
                    <DockPanel DockPanel.Dock="Top" Margin="0,0,0,6" LastChildFill="False">
                        <TextBlock Text="Sort by" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center" Margin="0,0,8,0"/>
                        <Button x:Name="BtnSortName"    Content="Name"    Tag="name"    Click="SortServices_Click" Style="{StaticResource SortBtn}"/>
                        <Button x:Name="BtnSortStatus"  Content="Status"  Tag="status"  Click="SortServices_Click" Style="{StaticResource SortBtn}"/>
                        <Button x:Name="BtnSortStartup" Content="Startup" Tag="startup" Click="SortServices_Click" Style="{StaticResource SortBtn}"/>
                    </DockPanel>
                    <TextBlock DockPanel.Dock="Bottom" x:Name="TxtSvcStatus" Foreground="#6C7086" FontSize="11" Margin="0,6,0,0"/>
                    <ListBox x:Name="LstServices" Style="{StaticResource PlainList}">
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <Border BorderBrush="#45475A" BorderThickness="0,0,0,1" Padding="0,6">
                                    <Grid>
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="*"/>
                                            <ColumnDefinition Width="90"/>
                                            <ColumnDefinition Width="Auto"/>
                                        </Grid.ColumnDefinitions>
                                        <StackPanel Grid.Column="0" VerticalAlignment="Center">
                                            <TextBlock Text="{Binding DisplayName}" Foreground="#CDD6F4" FontSize="12" TextTrimming="CharacterEllipsis"/>
                                            <TextBlock Foreground="#6C7086" FontSize="10">
                                                <Run Text="{Binding Name, Mode=OneWay}"/><Run Text="  ·  start: "/><Run Text="{Binding StartMode, Mode=OneWay}"/>
                                            </TextBlock>
                                        </StackPanel>
                                        <TextBlock Grid.Column="1" Text="{Binding State}" VerticalAlignment="Center" FontSize="11"
                                                   Foreground="#9399B2" TextAlignment="Center"/>
                                        <StackPanel Grid.Column="2" Orientation="Horizontal" VerticalAlignment="Center">
                                            <Button Content="Start"   Tag="{Binding Name}" Click="SvcStart_Click"   Style="{StaticResource SvcShowWhenStopped}" Padding="8,3" FontSize="10" Margin="2,0"/>
                                            <Button Content="Stop"    Tag="{Binding Name}" Click="SvcStop_Click"    Style="{StaticResource SvcShowWhenRunning}" Padding="8,3" FontSize="10" Margin="2,0"/>
                                            <Button Content="Restart" Tag="{Binding Name}" Click="SvcRestart_Click" Style="{StaticResource SvcShowWhenRunning}" Padding="8,3" FontSize="10" Margin="2,0"/>
                                        </StackPanel>
                                    </Grid>
                                </Border>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </DockPanel>
            </Border>

            <!-- SCHEDULED TASKS -->
            <Border x:Name="SecTasks" Style="{StaticResource Card}" Visibility="Collapsed">
                <DockPanel>
                    <TextBlock DockPanel.Dock="Top" x:Name="TxtTasksStatus" Foreground="#6C7086" FontSize="11" Margin="0,0,0,6"/>
                    <ListBox x:Name="LstTasks" Style="{StaticResource PlainList}">
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <Border BorderBrush="#45475A" BorderThickness="0,0,0,1" Padding="0,5">
                                    <Grid>
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="*"/><ColumnDefinition Width="90"/>
                                            <ColumnDefinition Width="150"/><ColumnDefinition Width="150"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Text="{Binding Name}" Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                        <TextBlock Grid.Column="1" Text="{Binding Status}" Foreground="#9399B2" FontSize="11"/>
                                        <TextBlock Grid.Column="2" Text="{Binding NextRun}" Foreground="#6C7086" FontSize="11"/>
                                        <TextBlock Grid.Column="3" Text="{Binding LastRun}" Foreground="#6C7086" FontSize="11"/>
                                    </Grid>
                                </Border>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </DockPanel>
            </Border>

            <!-- DRIVERS -->
            <Border x:Name="SecDrivers" Style="{StaticResource Card}" Visibility="Collapsed">
                <DockPanel>
                    <TextBlock DockPanel.Dock="Top" x:Name="TxtDriversStatus" Foreground="#6C7086" FontSize="11" Margin="0,0,0,6"/>
                    <ListBox x:Name="LstDrivers" Style="{StaticResource PlainList}">
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <Border BorderBrush="#45475A" BorderThickness="0,0,0,1" Padding="0,5">
                                    <Grid>
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="*"/><ColumnDefinition Width="160"/>
                                            <ColumnDefinition Width="110"/><ColumnDefinition Width="110"/><ColumnDefinition Width="80"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Text="{Binding Device}" Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                        <TextBlock Grid.Column="1" Text="{Binding Provider}" Foreground="#9399B2" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                        <TextBlock Grid.Column="2" Text="{Binding Version}" Foreground="#6C7086" FontSize="11"/>
                                        <TextBlock Grid.Column="3" Text="{Binding Date}" Foreground="#6C7086" FontSize="11"/>
                                        <TextBlock Grid.Column="4" Text="{Binding SignedText}" FontSize="11" TextAlignment="Right">
                                            <TextBlock.Style>
                                                <Style TargetType="TextBlock">
                                                    <Setter Property="Foreground" Value="#6C7086"/>
                                                    <Style.Triggers>
                                                        <DataTrigger Binding="{Binding Signed}" Value="False">
                                                            <Setter Property="Foreground" Value="#F38BA8"/>
                                                        </DataTrigger>
                                                    </Style.Triggers>
                                                </Style>
                                            </TextBlock.Style>
                                        </TextBlock>
                                    </Grid>
                                </Border>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </DockPanel>
            </Border>

            <!-- HOSTS FILE -->
            <Border x:Name="SecHosts" Style="{StaticResource Card}" Visibility="Collapsed">
                <DockPanel>
                    <DockPanel DockPanel.Dock="Top" Margin="0,0,0,8">
                        <Button DockPanel.Dock="Right" x:Name="BtnHostsSave" Content="Save" Style="{StaticResource ActionButton}" Click="HostsSave_Click"/>
                        <Button DockPanel.Dock="Right" x:Name="BtnHostsReload" Content="Reload" Style="{StaticResource ActionButton}" Margin="0,0,8,0" Click="HostsReload_Click"/>
                        <TextBlock Text="HOSTS FILE" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <TextBlock DockPanel.Dock="Bottom" x:Name="TxtHostsStatus" Foreground="#6C7086" FontSize="11" Margin="0,6,0,0"/>
                    <TextBox x:Name="TxtHosts" Background="#11111B" Foreground="#CDD6F4" FontFamily="Consolas" FontSize="12"
                             BorderThickness="0" Padding="8" AcceptsReturn="True" AcceptsTab="True"
                             VerticalScrollBarVisibility="Auto" TextWrapping="NoWrap"/>
                </DockPanel>
            </Border>

            <!-- ENVIRONMENT -->
            <Border x:Name="SecEnv" Style="{StaticResource Card}" Visibility="Collapsed">
                <DockPanel>
                    <!-- Add / update a variable -->
                    <StackPanel DockPanel.Dock="Top" Margin="0,0,0,10">
                        <TextBlock Text="ADD / UPDATE VARIABLE" Style="{StaticResource CardTitle}"/>
                        <Grid Margin="0,6,0,0">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="Auto"/>
                                <ColumnDefinition Width="200"/>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="Auto"/>
                            </Grid.ColumnDefinitions>
                            <Grid.RowDefinitions>
                                <RowDefinition Height="Auto"/>
                                <RowDefinition Height="Auto"/>
                            </Grid.RowDefinitions>

                            <TextBlock Grid.Row="0" Grid.Column="0" Text="Scope"  Foreground="#6C7086" FontSize="10" Margin="2,0,8,3"/>
                            <TextBlock Grid.Row="0" Grid.Column="1" Text="Name"   Foreground="#6C7086" FontSize="10" Margin="2,0,8,3"/>
                            <TextBlock Grid.Row="0" Grid.Column="2" Text="Value"  Foreground="#6C7086" FontSize="10" Margin="2,0,8,3"/>

                            <ComboBox Grid.Row="1" Grid.Column="0" x:Name="CmbEnvScope" Width="90" Height="30" Margin="0,0,8,0"
                                      VerticalContentAlignment="Center" SelectedIndex="0">
                                <ComboBoxItem Content="System"/>
                                <ComboBoxItem Content="User"/>
                            </ComboBox>
                            <Border Grid.Row="1" Grid.Column="1" Background="#45475A" CornerRadius="6" Padding="10,0" Margin="0,0,8,0">
                                <TextBox x:Name="TxtEnvName" Height="30" Background="Transparent" Foreground="#CDD6F4"
                                         FontSize="12" BorderThickness="0" VerticalContentAlignment="Center"
                                         ToolTip="Variable name (e.g. JAVA_HOME)"/>
                            </Border>
                            <Border Grid.Row="1" Grid.Column="2" Background="#45475A" CornerRadius="6" Padding="10,0" Margin="0,0,8,0">
                                <TextBox x:Name="TxtEnvValue" Height="30" Background="Transparent" Foreground="#CDD6F4"
                                         FontSize="12" BorderThickness="0" VerticalContentAlignment="Center"
                                         ToolTip="Value (e.g. C:\Program Files\Java\jdk)"/>
                            </Border>
                            <Button Grid.Row="1" Grid.Column="3" x:Name="BtnEnvAdd" Content="Add" Style="{StaticResource ActionButton}"
                                    Padding="16,4" Click="EnvAdd_Click"/>
                        </Grid>
                        <TextBlock x:Name="TxtEnvStatus" FontSize="11" Margin="0,6,0,0" TextWrapping="Wrap"/>
                        <TextBlock Text="System scope applies to all users and needs the change to be re-read by new processes (already-open apps keep the old value)."
                                   FontSize="10" Foreground="#6C7086" TextWrapping="Wrap" Margin="0,4,0,0"/>
                        <Rectangle Style="{StaticResource RowDivider}" Margin="0,10,0,0"/>
                    </StackPanel>

                    <ListBox x:Name="LstEnv" Style="{StaticResource PlainList}">
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <Border BorderBrush="#45475A" BorderThickness="0,0,0,1" Padding="0,5">
                                    <Grid>
                                        <Grid.ColumnDefinitions>
                                            <ColumnDefinition Width="70"/><ColumnDefinition Width="200"/><ColumnDefinition Width="*"/>
                                        </Grid.ColumnDefinitions>
                                        <TextBlock Grid.Column="0" Text="{Binding Scope}" Foreground="#6C7086" FontSize="11"/>
                                        <TextBlock Grid.Column="1" Text="{Binding Name}" Foreground="#CDD6F4" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                        <TextBlock Grid.Column="2" Text="{Binding Value}" Foreground="#9399B2" FontSize="11" TextTrimming="CharacterEllipsis"/>
                                    </Grid>
                                </Border>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </DockPanel>
            </Border>

            <!-- FEATURES -->
            <Border x:Name="SecFeatures" Style="{StaticResource Card}" Visibility="Collapsed">
                <DockPanel>
                    <DockPanel DockPanel.Dock="Top" Margin="0,0,0,8">
                        <Button DockPanel.Dock="Right" Content="Open Windows Features" Style="{StaticResource ActionButton}" Click="OpenFeatures_Click"/>
                        <TextBlock Text="WINDOWS OPTIONAL FEATURES" Style="{StaticResource CardTitle}" VerticalAlignment="Center"/>
                    </DockPanel>
                    <ListBox x:Name="LstFeatures" Style="{StaticResource PlainList}">
                        <ListBox.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,4">
                                    <Ellipse DockPanel.Dock="Left" Width="9" Height="9" VerticalAlignment="Center" Margin="0,0,10,0">
                                        <Ellipse.Style>
                                            <Style TargetType="Ellipse">
                                                <Setter Property="Fill" Value="#6C7086"/>
                                                <Style.Triggers>
                                                    <DataTrigger Binding="{Binding Enabled}" Value="True"><Setter Property="Fill" Value="#A6E3A1"/></DataTrigger>
                                                </Style.Triggers>
                                            </Style>
                                        </Ellipse.Style>
                                    </Ellipse>
                                    <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12"/>
                                </DockPanel>
                            </DataTemplate>
                        </ListBox.ItemTemplate>
                    </ListBox>
                </DockPanel>
            </Border>

            <!-- USER PROFILES -->
            <Border x:Name="SecProfiles" Style="{StaticResource Card}" Visibility="Collapsed">
                <StackPanel>
                    <TextBlock Text="USER PROFILES" Style="{StaticResource CardTitle}"/>
                    <ItemsControl x:Name="IcProfiles">
                        <ItemsControl.ItemTemplate>
                            <DataTemplate>
                                <DockPanel Margin="0,5">
                                    <Button DockPanel.Dock="Right" Content="Delete profile" Tag="{Binding}"
                                            Click="DeleteProfile_Click" Style="{StaticResource ActionButton}"
                                            Padding="10,4" FontSize="11" Margin="12,0,0,0" VerticalAlignment="Center"
                                            Visibility="{Binding CanDelete, Converter={StaticResource BoolToVis}}"/>
                                    <TextBlock DockPanel.Dock="Right" Foreground="#6C7086" FontSize="11" VerticalAlignment="Center">
                                        <Run Text="last used "/><Run Text="{Binding LastUsed, StringFormat='{}{0:d MMM yyyy}', Mode=OneWay}"/>
                                    </TextBlock>
                                    <StackPanel>
                                        <TextBlock Text="{Binding Name}" Foreground="#CDD6F4" FontSize="12" FontWeight="SemiBold"/>
                                        <TextBlock Text="{Binding Path}" Foreground="#6C7086" FontSize="11"/>
                                    </StackPanel>
                                </DockPanel>
                            </DataTemplate>
                        </ItemsControl.ItemTemplate>
                    </ItemsControl>
                    <TextBlock x:Name="TxtNoProfiles" Foreground="#6C7086" FontSize="12" Text="No non-system profiles found." Visibility="Collapsed"/>
                </StackPanel>
            </Border>

        </Grid>
    </Grid>
</UserControl>
```
