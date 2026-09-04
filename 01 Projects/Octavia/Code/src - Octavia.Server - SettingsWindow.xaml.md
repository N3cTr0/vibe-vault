---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\SettingsWindow.xaml
---

# src\Octavia.Server\SettingsWindow.xaml

```xml
<Window x:Class="Octavia.Server.SettingsWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Octavia — her server" Height="660" Width="720"
        WindowStartupLocation="CenterScreen" Background="#F4F3F1">

  <Window.Resources>
    <Style TargetType="TextBlock" x:Key="Label">
      <Setter Property="VerticalAlignment" Value="Center" />
      <Setter Property="Margin" Value="0,0,12,0" />
      <Setter Property="Foreground" Value="#3A3733" />
    </Style>
    <Style TargetType="TextBlock" x:Key="Hint">
      <Setter Property="Foreground" Value="#7A736B" />
      <Setter Property="FontSize" Value="11" />
      <Setter Property="TextWrapping" Value="Wrap" />
      <Setter Property="Margin" Value="0,2,0,10" />
    </Style>
    <Style TargetType="TextBlock" x:Key="Head">
      <Setter Property="FontWeight" Value="SemiBold" />
      <Setter Property="Margin" Value="0,14,0,6" />
    </Style>
    <Style TargetType="TextBox">
      <Setter Property="Padding" Value="4,3" />
      <Setter Property="Margin" Value="0,0,0,2" />
    </Style>
    <Style TargetType="ComboBox">
      <Setter Property="Padding" Value="4,3" />
      <Setter Property="Margin" Value="0,0,0,2" />
    </Style>
    <Style TargetType="CheckBox">
      <Setter Property="Margin" Value="0,4,0,2" />
    </Style>
  </Window.Resources>

  <DockPanel Margin="0">

    <Border DockPanel.Dock="Top" Background="#FFFFFF" Padding="18,14" BorderBrush="#E2DED8" BorderThickness="0,0,0,1">
      <StackPanel>
        <TextBlock Text="Octavia" FontSize="20" FontWeight="Bold" />
        <TextBlock x:Name="StateLine" Style="{StaticResource Hint}" Margin="0,3,0,0" />
      </StackPanel>
    </Border>

    <Border DockPanel.Dock="Bottom" Background="#FFFFFF" Padding="18,12" BorderBrush="#E2DED8" BorderThickness="0,1,0,0">
      <DockPanel>
        <TextBlock x:Name="SaveNote" DockPanel.Dock="Left" Style="{StaticResource Hint}"
                   VerticalAlignment="Center" Margin="0" MaxWidth="380" />
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Right">
          <Button x:Name="CloseButton" Content="Close" Padding="16,6" Margin="0,0,8,0" Click="OnClose" />
          <Button x:Name="SaveButton" Content="Save" Padding="16,6" Margin="0,0,8,0" Click="OnSave" />
          <Button x:Name="SaveRestartButton" Content="Save and restart her" Padding="16,6" Click="OnSaveAndRestart" />
        </StackPanel>
      </DockPanel>
    </Border>

    <TabControl Margin="12" Background="Transparent" BorderThickness="0">

      <TabItem Header="Brain">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock Text="Which brain" Style="{StaticResource Head}" />
            <ComboBox x:Name="BrainBox">
              <ComboBoxItem Content="local" />
              <ComboBoxItem Content="claude" />
            </ComboBox>
            <TextBlock Style="{StaticResource Hint}"
                       Text="Local costs nothing, needs no key and works with the network down. Claude is the hosted model." />

            <!-- Second on the tab, and it used to be last. A badge saying a key is stored is
                 worth nothing under four settings and a scrollbar — the window is 660 tall and
                 this sat below the fold, which is indistinguishable from the window not saying
                 it at all. That was the actual complaint; the colour was only half the fix. -->
            <TextBlock Text="Anthropic API key" Style="{StaticResource Head}" />
            <StackPanel Orientation="Horizontal">
              <PasswordBox x:Name="ApiKeyBox" Width="300" Padding="4,3" />
              <Button Content="Store" Padding="12,4" Margin="8,0,0,0" Click="OnStoreKey" />
              <Button Content="Clear" Padding="12,4" Margin="6,0,0,0" Click="OnClearKey" />
              <!-- The same badge an integration's password gets. It was a grey hint below
                   the buttons and read as one more caption; whether a credential is stored
                   is a state, and states get a colour. -->
              <TextBlock x:Name="KeyState" VerticalAlignment="Center" Margin="10,0,0,0" FontWeight="SemiBold" />
            </StackPanel>
            <TextBlock x:Name="KeyNote" Style="{StaticResource Hint}" />

            <TextBlock Text="Claude model" Style="{StaticResource Head}" />
            <TextBox x:Name="ModelBox" />

            <TextBlock Text="Local model" Style="{StaticResource Head}" />
            <TextBox x:Name="LocalModelBox" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Any model your OpenAI-compatible server has. Wall-clock beats tokens per second: a chattier model that ignores 'be brief' finishes later than a slower one that stops talking." />

            <TextBlock Text="Local endpoint" Style="{StaticResource Head}" />
            <TextBox x:Name="LocalEndpointBox" />
            <TextBlock Style="{StaticResource Hint}" Text="Ollama, LM Studio, llama-server — anything OpenAI-compatible." />

            <TextBlock Text="Reply length (max tokens)" Style="{StaticResource Head}" />
            <TextBox x:Name="MaxTokensBox" Width="120" HorizontalAlignment="Left" />
          </StackPanel>
        </ScrollViewer>
      </TabItem>

      <TabItem Header="Ears">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock Text="Speech model" Style="{StaticResource Head}" />
            <ComboBox x:Name="WhisperModelBox" IsEditable="True" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="large-v3-turbo is the conversation default; large-v3 when accuracy beats latency; small.en on a weak machine. Downloaded on first use." />

            <TextBlock Text="Language" Style="{StaticResource Head}" />
            <TextBox x:Name="WhisperLanguageBox" Width="120" HorizontalAlignment="Left" />
            <TextBlock Style="{StaticResource Hint}" Text="An ISO code, or 'auto' to detect per utterance." />

            <TextBlock Text="Compute" Style="{StaticResource Head}" />
            <ComboBox x:Name="WhisperComputeBox">
              <ComboBoxItem Content="auto" />
              <ComboBoxItem Content="gpu" />
              <ComboBoxItem Content="cpu" />
            </ComboBox>

            <TextBlock Text="Wake phrase" Style="{StaticResource Head}" />
            <TextBox x:Name="WakePhraseBox" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Empty means no wake word. Deliberately empty until hers is trained — pointing it at somebody else's phrase means she answers to a name that is not hers." />

            <TextBlock Text="Wake threshold" Style="{StaticResource Head}" />
            <TextBox x:Name="WakeThresholdBox" Width="120" HorizontalAlignment="Left" />

            <CheckBox x:Name="OpenEarsBox" Content="Load the speech models when she starts" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Costs about 1.6 GB of memory from boot and buys back the first sentence." />
          </StackPanel>
        </ScrollViewer>
      </TabItem>

      <TabItem Header="Voice and face">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock Text="Voice" Style="{StaticResource Head}" />
            <TextBlock x:Name="VoiceLine" Style="{StaticResource Hint}" />

            <TextBlock Text="Speaking rate" Style="{StaticResource Head}" />
            <StackPanel Orientation="Horizontal">
              <Slider x:Name="VoiceRateSlider" Minimum="-10" Maximum="10" Width="300"
                      TickFrequency="1" IsSnapToTickEnabled="True" VerticalAlignment="Center" />
              <TextBlock x:Name="VoiceRateValue" Style="{StaticResource Label}" Margin="12,0,0,0" />
            </StackPanel>
            <TextBlock Style="{StaticResource Hint}" Text="Rising is faster. Zero is how she was recorded." />

            <TextBlock Text="Avatar" Style="{StaticResource Head}" />
            <ComboBox x:Name="AvatarBox" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Drop a .vrm in her avatars folder to add one. The plaster bust is the fallback that always works." />

            <TextBlock Text="Room lighting hour" Style="{StaticResource Head}" />
            <TextBox x:Name="RoomHourBox" Width="120" HorizontalAlignment="Left" />
            <TextBlock Style="{StaticResource Hint}" Text="0–23 pins the light. Negative follows the real clock." />

            <CheckBox x:Name="ShowStatsBox" Content="Show the status readout over her corner" />
            <CheckBox x:Name="ShowCaptionBox" Content="Show the captions over the room" />
            <CheckBox x:Name="CameraBox" Content="Allow her to open a camera when a question needs eyes" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Off by default, and the only sense that is: a microphone in a room is expected, a camera is not. One still, never a watch." />
          </StackPanel>
        </ScrollViewer>
      </TabItem>

      <TabItem Header="Rounds">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock x:Name="RoundsState" Style="{StaticResource Hint}" Margin="0,0,0,8" />

            <CheckBox x:Name="RoundsEnabledBox" Content="Let her check things on her own" />

            <TextBlock Text="Minutes between checks" Style="{StaticResource Head}" />
            <TextBox x:Name="RoundsEveryBox" Width="120" HorizontalAlignment="Left" />

            <TextBlock Text="Days spent learning what is normal" Style="{StaticResource Head}" />
            <TextBox x:Name="LearnDaysBox" Width="120" HorizontalAlignment="Left" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="She says nothing at all until this has passed, counted from her first check rather than from installation. Nothing about your network is written into her: she learns what is normal wherever she is installed. Changing this does not restart the learning." />

            <TextBlock Text="Quiet from / until" Style="{StaticResource Head}" />
            <StackPanel Orientation="Horizontal">
              <TextBox x:Name="QuietFromBox" Width="90" />
              <TextBlock Text="to" Style="{StaticResource Label}" Margin="10,0" />
              <TextBox x:Name="QuietToBox" Width="90" />
            </StackPanel>
            <TextBlock Style="{StaticResource Hint}"
                       Text="24-hour times. Anything she finds inside this window is held rather than dropped, and said when it ends. Equal times mean no quiet hours." />
          </StackPanel>
        </ScrollViewer>
      </TabItem>

      <TabItem Header="Integrations">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock Style="{StaticResource Hint}" Margin="0,0,0,10"
                       Text="Each of these is a separate process she starts and calls. Settings live in the environment it is given; passwords are sealed to your Windows account and are never written into config.json." />
            <StackPanel x:Name="IntegrationsPanel" />
          </StackPanel>
        </ScrollViewer>
      </TabItem>

      <TabItem Header="Server and logs">
        <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="14">
          <StackPanel>
            <TextBlock Text="Profile" Style="{StaticResource Head}" />
            <ComboBox x:Name="ProfileBox" IsEditable="True" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="A named set of overrides. The server's own --profile argument wins over this." />

            <TextBlock Text="Face port" Style="{StaticResource Head}" />
            <TextBox x:Name="FacePortBox" Width="120" HorizontalAlignment="Left" />

            <CheckBox x:Name="RemoteAccessBox" Content="Let faces on the network reach her" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="Off binds to this machine only. On binds every interface, and a remote face must present the remote key. Reach it over Tailscale or Wireguard, never a forwarded port." />

            <TextBlock Text="Log level" Style="{StaticResource Head}" />
            <ComboBox x:Name="LogLevelBox">
              <ComboBoxItem Content="debug" />
              <ComboBoxItem Content="info" />
              <ComboBoxItem Content="warn" />
              <ComboBoxItem Content="error" />
            </ComboBox>

            <TextBlock Text="Days of logs to keep" Style="{StaticResource Head}" />
            <TextBox x:Name="LogKeepBox" Width="120" HorizontalAlignment="Left" />
            <TextBlock Style="{StaticResource Hint}"
                       Text="One file per day. Older ones are deleted after midnight. Zero keeps everything." />

            <StackPanel Orientation="Horizontal" Margin="0,16,0,0">
              <Button Content="Open data folder" Padding="12,5" Click="OnOpenData" />
              <Button Content="Open her log" Padding="12,5" Margin="8,0,0,0" Click="OnOpenLog" />
            </StackPanel>
          </StackPanel>
        </ScrollViewer>
      </TabItem>

    </TabControl>
  </DockPanel>
</Window>
```
