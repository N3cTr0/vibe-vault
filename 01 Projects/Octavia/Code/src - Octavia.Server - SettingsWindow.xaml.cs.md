---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\SettingsWindow.xaml.cs
---

# src\Octavia.Server\SettingsWindow.xaml.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Windows;
using System.Windows.Controls;
using Octavia.Core;
using Octavia.Voice;
using Button = System.Windows.Controls.Button;
using ComboBox = System.Windows.Controls.ComboBox;
using CheckBox = System.Windows.Controls.CheckBox;
using MessageBox = System.Windows.MessageBox;
using Orientation = System.Windows.Controls.Orientation;
using TextBox = System.Windows.Controls.TextBox;

namespace Octavia.Server;

/// Everything about her server that is a setting, in one window.
///
/// **It edits the file, not a running session.** `config.json` and the sealed secret store are
/// the two things the server reads at startup, so writing them is the whole of configuring
/// her — and it keeps working while she is stopped, which is exactly when somebody needs to
/// change the setting that is stopping her.
///
/// `OctaviaConfig.Save` does the careful part: it writes only the keys that actually changed,
/// into the *base* of the file, so a profile overlay is never baked in. That was a real
/// regression once and the merge exists because of it.
public partial class SettingsWindow : Window
{
    private readonly OctaviaConfig _config;

    /// Keys the active profile overrides. Editing one of those here writes a value the server
    /// will then ignore, which is the single most confusing thing this window could do
    /// quietly — so it is said out loud instead.
    private readonly HashSet<string> _overridden;

    public SettingsWindow()
    {
        InitializeComponent();

        _config = OctaviaConfig.Load();
        _overridden = OverriddenKeys(_config);

        Load();
        BuildIntegrations();
    }

    private static HashSet<string> OverriddenKeys(OctaviaConfig config)
    {
        var set = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

        if (config.Profiles.TryGetValue(config.Profile, out var profile))
            foreach (var key in profile.Select(pair => pair.Key)) set.Add(key);

        return set;
    }

    private void Load()
    {
        var state = ServerControl.ServiceState switch
        {
            0 => "Running",
            2 => "Stopped",
            1 => "No service installed",
            _ => "Octavia.Server.exe was not found"
        };

        var note = _overridden.Count > 0
            ? $"  ·  profile '{_config.Profile}' overrides {string.Join(", ", _overridden.Order())}, " +
              "so changing those here will not take effect"
            : "";

        StateLine.Text = $"{state}  ·  {Paths.DataDir}{note}";

        Select(BrainBox, _config.Brain);
        LocalModelBox.Text = _config.LocalModel;
        LocalEndpointBox.Text = _config.LocalEndpoint;
        ModelBox.Text = _config.Model;
        MaxTokensBox.Text = _config.MaxTokens.ToString();
        KeyNote.Text = SecretStore.ReadApiKey() is { Length: > 0 }
            ? "A key is stored, sealed to this Windows account. It is never shown again."
            : "No key stored. Only needed for the Claude brain.";

        foreach (var model in new[] { "tiny.en", "base.en", "small.en", "medium.en", "large-v3", "large-v3-turbo" })
            WhisperModelBox.Items.Add(model);
        WhisperModelBox.Text = _config.WhisperModel;

        WhisperLanguageBox.Text = _config.WhisperLanguage;
        Select(WhisperComputeBox, _config.WhisperCompute);
        WakePhraseBox.Text = _config.WakePhrase;
        WakeThresholdBox.Text = _config.WakeThreshold.ToString("0.00");
        OpenEarsBox.IsChecked = _config.OpenEarsOnStart;

        // She has one voice and it is not a setting — Stage 16. Said rather than offered as a
        // menu of one, which is the fault this project keeps writing down.
        VoiceLine.Text = $"Kokoro ({KokoroStore.Voice}). She has one voice, chosen by ear out of " +
                         "twenty-two, and there is nothing to pick between.";

        VoiceRateSlider.Value = _config.VoiceRate;
        VoiceRateValue.Text = _config.VoiceRate.ToString();
        VoiceRateSlider.ValueChanged += (_, e) => VoiceRateValue.Text = ((int)e.NewValue).ToString();

        AvatarBox.Items.Add("Plaster bust");
        foreach (var file in Avatars()) AvatarBox.Items.Add(file);
        AvatarBox.SelectedItem = string.IsNullOrEmpty(_config.AvatarFile) ? "Plaster bust" : _config.AvatarFile;
        if (AvatarBox.SelectedItem is null) AvatarBox.SelectedIndex = 0;

        RoomHourBox.Text = _config.RoomHour.ToString();
        ShowStatsBox.IsChecked = _config.ShowStats;
        CameraBox.IsChecked = _config.Camera;

        RoundsEnabledBox.IsChecked = _config.Rounds.Enabled;
        RoundsEveryBox.Text = _config.Rounds.EveryMinutes.ToString();
        LearnDaysBox.Text = _config.Rounds.LearnForDays.ToString();
        QuietFromBox.Text = _config.Rounds.QuietFrom;
        QuietToBox.Text = _config.Rounds.QuietTo;
        RoundsState.Text = Learning();

        foreach (var name in _config.Profiles.Keys) ProfileBox.Items.Add(name);
        ProfileBox.Text = _config.Profile;
        FacePortBox.Text = _config.FacePort.ToString();
        RemoteAccessBox.IsChecked = _config.RemoteAccess;
        Select(LogLevelBox, _config.LogLevel);
        LogKeepBox.Text = _config.LogKeepDays.ToString();
    }

    /// How far through the learning she is, read from the baseline file rather than asked of
    /// her — this window works while she is stopped, and that is when somebody is most likely
    /// to be wondering why she has never said anything.
    private string Learning()
    {
        var file = Path.Combine(Paths.DataDir, "baseline-threats.json");
        if (!File.Exists(file)) return "She has not watched anything yet.";

        try
        {
            using var json = System.Text.Json.JsonDocument.Parse(File.ReadAllText(file));

            if (!json.RootElement.TryGetProperty("Began", out var began) ||
                !began.TryGetDateTimeOffset(out var start))
                return "She has not watched anything yet.";

            var done = DateTimeOffset.Now - start;
            var left = TimeSpan.FromDays(_config.Rounds.LearnForDays) - done;

            return left <= TimeSpan.Zero
                ? $"Learning finished. She has been watching since {start.LocalDateTime:d MMMM}."
                : $"Learning since {start.LocalDateTime:d MMMM} — about {left.TotalDays:0.#} days to go.";
        }
        catch (Exception ex)
        {
            return $"Could not read what she has learned: {ex.Message}";
        }
    }

    /// One block per configured tool server: whether it is on, the values it is given, and a
    /// button per secret. Built rather than declared, because the set of integrations is
    /// whatever `config.json` holds and grows without this window being touched.
    private void BuildIntegrations()
    {
        if (_config.McpServers.Count == 0)
        {
            IntegrationsPanel.Children.Add(new TextBlock
            {
                Text = "No integrations are configured.",
                Foreground = System.Windows.Media.Brushes.Gray
            });
            return;
        }

        foreach (var (name, server) in _config.McpServers.OrderBy(pair => pair.Key))
        {
            var box = new StackPanel { Margin = new Thickness(0, 0, 0, 18) };

            box.Children.Add(new TextBlock
            {
                Text = name,
                FontWeight = FontWeights.SemiBold,
                FontSize = 14,
                Margin = new Thickness(0, 0, 0, 4)
            });

            var enabled = new CheckBox { Content = "Enabled", IsChecked = server.Enabled };
            enabled.Checked += (_, _) => server.Enabled = true;
            enabled.Unchecked += (_, _) => server.Enabled = false;
            box.Children.Add(enabled);

            foreach (var key in (server.Env ?? []).Keys.OrderBy(k => k).ToList())
            {
                var row = new DockPanel { Margin = new Thickness(0, 4, 0, 0) };

                row.Children.Add(new TextBlock
                {
                    Text = key,
                    Width = 170,
                    VerticalAlignment = VerticalAlignment.Center,
                    Foreground = System.Windows.Media.Brushes.DimGray
                });

                var value = new TextBox { Text = server.Env![key] };
                value.TextChanged += (_, _) => server.Env[key] = value.Text;
                row.Children.Add(value);

                box.Children.Add(row);
            }

            foreach (var secret in server.Secrets ?? [])
            {
                var held = SecretStore.HasFor(name, secret);

                var row = new StackPanel { Orientation = Orientation.Horizontal, Margin = new Thickness(0, 6, 0, 0) };

                row.Children.Add(new TextBlock
                {
                    Text = secret,
                    Width = 170,
                    VerticalAlignment = VerticalAlignment.Center,
                    Foreground = System.Windows.Media.Brushes.DimGray
                });

                var entry = new PasswordBox { Width = 220, Padding = new Thickness(4, 3, 4, 3) };
                row.Children.Add(entry);

                var status = new TextBlock
                {
                    Text = held ? "  stored" : "  not set",
                    VerticalAlignment = VerticalAlignment.Center,
                    Foreground = held ? System.Windows.Media.Brushes.SeaGreen : System.Windows.Media.Brushes.Crimson
                };

                var store = new Button { Content = "Store", Padding = new Thickness(10, 3, 10, 3), Margin = new Thickness(8, 0, 0, 0) };
                store.Click += (_, _) =>
                {
                    if (entry.Password.Length == 0) return;

                    SecretStore.WriteFor(name, secret, entry.Password);
                    entry.Clear();
                    status.Text = "  stored";
                    status.Foreground = System.Windows.Media.Brushes.SeaGreen;
                    SaveNote.Text = $"{secret} stored. Restart her for it to be used.";
                };

                var clear = new Button { Content = "Clear", Padding = new Thickness(10, 3, 10, 3), Margin = new Thickness(6, 0, 0, 0) };
                clear.Click += (_, _) =>
                {
                    SecretStore.ClearFor(name, secret);
                    status.Text = "  not set";
                    status.Foreground = System.Windows.Media.Brushes.Crimson;
                };

                row.Children.Add(store);
                row.Children.Add(clear);
                row.Children.Add(status);

                box.Children.Add(row);
            }

            box.Children.Add(new TextBlock
            {
                Text = server.Command + " " + string.Join(' ', server.Args ?? []),
                FontSize = 11,
                Foreground = System.Windows.Media.Brushes.Gray,
                TextWrapping = TextWrapping.Wrap,
                Margin = new Thickness(0, 6, 0, 0)
            });

            IntegrationsPanel.Children.Add(box);
        }
    }

    private static IEnumerable<string> Avatars()
    {
        try { return Directory.EnumerateFiles(Paths.AvatarDir, "*.vrm").Select(Path.GetFileName)!; }
        catch { return []; }
    }

    private static void Select(ComboBox box, string? value)
    {
        foreach (var item in box.Items)
            if (item is ComboBoxItem entry &&
                string.Equals(entry.Content?.ToString(), value, StringComparison.OrdinalIgnoreCase))
            {
                box.SelectedItem = entry;
                return;
            }

        if (box.IsEditable) box.Text = value ?? "";
        else if (box.Items.Count > 0) box.SelectedIndex = 0;
    }

    private static string Chosen(ComboBox box) =>
        box.SelectedItem is ComboBoxItem item ? item.Content?.ToString() ?? "" : box.Text;

    /// Reads every control back into the config. **Nothing is written on a failed parse**: a
    /// mistyped port silently becoming 0 is worse than being told about it.
    private bool Collect()
    {
        var trouble = new List<string>();

        _config.Brain = Chosen(BrainBox);
        _config.LocalModel = LocalModelBox.Text.Trim();
        _config.LocalEndpoint = LocalEndpointBox.Text.Trim();
        _config.Model = ModelBox.Text.Trim();
        _config.MaxTokens = Number(MaxTokensBox.Text, _config.MaxTokens, "reply length", trouble);

        _config.WhisperModel = WhisperModelBox.Text.Trim();
        _config.WhisperLanguage = WhisperLanguageBox.Text.Trim();
        _config.WhisperCompute = Chosen(WhisperComputeBox);
        _config.WakePhrase = WakePhraseBox.Text.Trim();

        if (double.TryParse(WakeThresholdBox.Text, out var threshold)) _config.WakeThreshold = threshold;
        else trouble.Add("wake threshold");

        _config.OpenEarsOnStart = OpenEarsBox.IsChecked == true;

        _config.VoiceRate = (int)VoiceRateSlider.Value;
        _config.AvatarFile = AvatarBox.SelectedItem is string chosen && chosen != "Plaster bust" ? chosen : "";
        _config.RoomHour = Number(RoomHourBox.Text, _config.RoomHour, "room hour", trouble);
        _config.ShowStats = ShowStatsBox.IsChecked == true;
        _config.Camera = CameraBox.IsChecked == true;

        _config.Rounds.Enabled = RoundsEnabledBox.IsChecked == true;
        _config.Rounds.EveryMinutes = Number(RoundsEveryBox.Text, _config.Rounds.EveryMinutes, "minutes between checks", trouble);
        _config.Rounds.LearnForDays = Number(LearnDaysBox.Text, _config.Rounds.LearnForDays, "learning days", trouble);
        _config.Rounds.QuietFrom = QuietFromBox.Text.Trim();
        _config.Rounds.QuietTo = QuietToBox.Text.Trim();

        _config.Profile = ProfileBox.Text.Trim();
        _config.FacePort = Number(FacePortBox.Text, _config.FacePort, "face port", trouble);
        _config.RemoteAccess = RemoteAccessBox.IsChecked == true;
        _config.LogLevel = Chosen(LogLevelBox);
        _config.LogKeepDays = Number(LogKeepBox.Text, _config.LogKeepDays, "days of logs", trouble);

        if (trouble.Count == 0) return true;

        MessageBox.Show(this,
            $"These are not numbers, so nothing was saved:\n\n  {string.Join("\n  ", trouble)}",
            "Octavia", MessageBoxButton.OK, MessageBoxImage.Warning);

        return false;
    }

    private static int Number(string text, int fallback, string what, List<string> trouble)
    {
        if (int.TryParse(text.Trim(), out var parsed)) return parsed;
        trouble.Add(what);
        return fallback;
    }

    private void OnSave(object sender, RoutedEventArgs e)
    {
        if (!Save()) return;
        SaveNote.Text = "Saved. Most of this takes effect when she restarts.";
    }

    private void OnSaveAndRestart(object sender, RoutedEventArgs e)
    {
        if (!Save()) return;

        SaveNote.Text = "Saved. Restarting her...";
        SaveRestartButton.IsEnabled = false;

        Task.Run(() =>
        {
            var worked = ServerControl.Restart();

            Dispatcher.Invoke(() =>
            {
                SaveNote.Text = worked ? "Saved, and she is running again." : "Saved, but the restart failed — her log says why.";
                SaveRestartButton.IsEnabled = true;
                StateLine.Text = StateLine.Text;
            });
        });
    }

    private bool Save()
    {
        if (!Collect()) return false;

        try
        {
            _config.Save();
            return true;
        }
        catch (Exception ex)
        {
            MessageBox.Show(this, $"Could not write config.json:\n\n{ex.Message}",
                "Octavia", MessageBoxButton.OK, MessageBoxImage.Error);
            return false;
        }
    }

    private void OnStoreKey(object sender, RoutedEventArgs e)
    {
        if (ApiKeyBox.Password.Length == 0) return;

        SecretStore.WriteApiKey(ApiKeyBox.Password);
        ApiKeyBox.Clear();
        KeyNote.Text = "A key is stored, sealed to this Windows account. It is never shown again.";
    }

    private void OnClearKey(object sender, RoutedEventArgs e)
    {
        SecretStore.ClearApiKey();
        KeyNote.Text = "No key stored. Only needed for the Claude brain.";
    }

    private void OnOpenData(object sender, RoutedEventArgs e) =>
        Process.Start("explorer.exe", Paths.DataDir);

    private void OnOpenLog(object sender, RoutedEventArgs e)
    {
        var log = Log.Today;
        if (!File.Exists(log)) { SaveNote.Text = "There is no log for today yet."; return; }

        Process.Start(new ProcessStartInfo(log) { UseShellExecute = true });
    }

    private void OnClose(object sender, RoutedEventArgs e) => Close();
}
```
