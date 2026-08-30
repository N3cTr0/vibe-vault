---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\OctaviaSession.cs
---

# src\Octavia.App\OctaviaSession.cs

```csharp
using System.Text;
using System.Text.Json;
using Octavia.Core;
using Octavia.Brain;
using Octavia.Diagnostics;
using Octavia.Face;
using Octavia.Senses;
using Octavia.Senses.Music;
using Octavia.Voice;

namespace Octavia;

/// Wires the four pieces together: what she hears, what she thinks, what she says,
/// and what the face is told about any of it.
internal sealed class OctaviaSession : IDisposable
{
    private readonly OctaviaConfig _config;
    private readonly IFaceTransport _face;
    private readonly IBrain _brain;
    private IVoice _voice;
    private readonly MicLevelMeter _meter = new();
    private readonly MusicWatcher _music = new();
    private readonly AttentionGate _gate;

    private ISpeechRecognizer? _ears;
    private bool _earsOpening;
    private CancellationTokenSource? _turn;
    private AgentState _state = AgentState.Idle;
    private bool _responding;
    private bool _wantsToListen;
    private bool _faceBuilt;
    private Mood _mood = Mood.Neutral;
    private float _lastSentLevel = -1f;
    private bool _disposed;

    public OctaviaSession(OctaviaConfig config, IFaceTransport face)
    {
        _config = config;
        _face = face;
        _gate = new AttentionGate(config);
        _meter.Device = config.MicrophoneDevice;
        _music.Device = config.OutputDevice;
        _brain = string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase)
            ? new LocalBrain(config)
            : new ClaudeBrain(config);
        Log.Write($"brain: {_brain.Description}");
        _voice = new SapiVoice(config);
        Listen(_voice);

        // The neural engine may have to download 80 MB before it can say anything, so
        // she starts with the Windows voice and upgrades herself when it is ready. That
        // way a first run talks immediately instead of sitting mute.
        if (string.Equals(config.VoiceEngine, "neural", StringComparison.OrdinalIgnoreCase))
            UseNeuralVoice().Forget("starting the neural voice");

        _face.MessageReceived += OnFaceMessage;

        _meter.LevelChanged += level =>
        {
            if (Math.Abs(level - _lastSentLevel) < 0.02f) return;
            _lastSentLevel = level;
            _face.Send(new { type = "level", value = level });
        };

        _music.Changed += state => _face.Send(new
        {
            type = "music",
            playing = state.Playing,
            bpm = state.Bpm,
            energy = state.Energy,
            beat = false
        });

        // The beat is sent on its own and never coalesced with the state above: a beat
        // that arrives 200 ms late is worse than one that never arrived at all.
        _music.Beat += () => _face.Send(new { type = "music", beat = true });

        if (_config.Music) StartMusic().Forget("starting to listen for music");
    }

    // ---- what the machine is playing --------------------------

    private async Task StartMusic()
    {
        if (!await _music.StartAsync())
        {
            // Not a failure worth interrupting for: plenty of machines have no render
            // endpoint they can tap, and she works perfectly well without ever hearing
            // one. It is in the self-test for whoever goes looking.
            Log.Warn("music: nothing to listen to, so she will not hear what is playing");
        }

        // The answer arrives after the first hello, so the face is told again.
        Announce();
    }

    private void SelectMusic(bool on)
    {
        _config.Music = on;
        _config.Save();

        Log.Write($"music listening {(on ? "on" : "off")}");

        if (on) StartMusic().Forget("starting to listen for music");
        else { _music.Stop(); Announce(); }
    }

    // ---- which devices she uses -------------------------------

    private static string Named(string? device) =>
        string.IsNullOrWhiteSpace(device) ? "the Windows default" : $"'{device}'";

    /// Changing the microphone reopens the ears, because a capture device is chosen
    /// when the stream opens and not afterwards. She keeps listening across the swap
    /// if she was listening before it.
    private void SelectMicrophone(string device)
    {
        _config.MicrophoneDevice = device;
        _config.Save();
        Log.Write($"microphone: {Named(device)}");

        _meter.Device = device;
        if (_meter.IsRunning) { _meter.Stop(); _meter.Start(); }

        if (_ears is not null)
        {
            var wasListening = _wantsToListen;
            StopListening();
            _ears.Dispose();
            _ears = null;
            if (wasListening) StartListening();
        }

        Announce();
    }

    /// The loopback capture is pinned to one endpoint, so this restarts it rather than
    /// letting it follow anything.
    private void SelectOutput(string device)
    {
        _config.OutputDevice = device;
        _config.Save();
        Log.Write($"output: {Named(device)}");

        _music.Device = device;
        if (_config.Music)
        {
            _music.Stop();
            StartMusic().Forget("restarting music after an output change");
        }
        else Announce();
    }

    /// Whisper.net binds its native library once per process, so this cannot take
    /// effect until she is restarted — and saying so is better than appearing to
    /// change something that has not changed.
    private void SelectWhisperCompute(string compute)
    {
        var wanted = compute.Trim().ToLowerInvariant();
        if (wanted is not ("auto" or "cpu" or "gpu")) return;

        _config.WhisperCompute = wanted;
        _config.Save();
        Log.Write($"whisper compute set to {wanted}; takes effect on restart");
        Notice($"Whisper will use {wanted} the next time she starts.");
        Announce();
    }

    private void Listen(IVoice voice)
    {
        voice.Viseme += (openness, shape) => _face.Send(new { type = "viseme", value = openness, shape });
        voice.Started += () => SetState(AgentState.Speaking);
        voice.Finished += OnVoiceFinished;
        voice.Trouble += Notice;
    }

    // ---- her voice -------------------------------------------

    /// Swaps the engine under a running session. Everything downstream — the face, the
    /// state machine, the mouth — is fed through `IVoice`, so neither knows it happened.
    private async Task UseNeuralVoice()
    {
        var neural = new NeuralVoice(_config);
        Listen(neural);

        try
        {
            await neural.StartAsync(_config.NeuralVoiceName);
        }
        catch (Exception ex)
        {
            Log.Error("the neural voice would not start", ex);
            Notice("Her neural voice could not start; staying with the Windows one.");
            neural.Dispose();
            return;
        }

        var previous = _voice;
        previous.Hush();
        _voice = neural;
        previous.Dispose();

        Log.Write($"voice engine: {neural.EngineName}");
        Announce();
    }

    private void UseWindowsVoice()
    {
        if (_voice is SapiVoice) return;

        var sapi = new SapiVoice(_config);
        Listen(sapi);

        var previous = _voice;
        previous.Hush();
        _voice = sapi;
        previous.Dispose();

        Log.Write($"voice engine: {sapi.EngineName}");
        Announce();
    }

    private void SelectVoiceEngine(string? engine)
    {
        var neural = string.Equals(engine, "neural", StringComparison.OrdinalIgnoreCase);
        _config.VoiceEngine = neural ? "neural" : "windows";
        _config.Save();

        if (neural) UseNeuralVoice().Forget("switching to the neural voice");
        else UseWindowsVoice();
    }

    // ---- face to host ----------------------------------------

    private void OnFaceMessage(JsonElement message)
    {
        if (!message.TryGetProperty("type", out var typeNode)) return;

        switch (typeNode.GetString())
        {
            case "ready":
                _faceBuilt = message.TryGetProperty("faceBuilt", out var b) && b.ValueKind == JsonValueKind.True;
                if (_faceBuilt) Log.Write("face ready; scene built");
                else Log.Warn("face ready but the scene did not build - check WebGL support and the browser console");
                Announce();
                if (_config.ListenOnStart) StartListening();
                break;

            case "faceError":
                Log.Error($"face error: {Text(message, "text")}");
                break;

            case "say":
                var text = Text(message, "text");
                if (!string.IsNullOrWhiteSpace(text)) RespondTo(text.Trim()).Forget("turn");
                break;

            case "listen":
                ToggleListening();
                break;

            case "hush":
                Hush();
                break;

            case "forget":
                _brain.Forget();
                _face.Send(new { type = "cleared" });
                Notice("Conversation forgotten.");
                break;

            case "setKey":
                SaveKey(Text(message, "value"));
                break;

            case "setVoice":
                var voice = Text(message, "value");
                if (_voice.SelectVoice(voice))
                {
                    if (_voice is NeuralVoice) _config.NeuralVoiceName = voice ?? "";
                    else _config.VoiceName = voice;
                    _config.Save();
                    Announce();
                }
                else if (_voice is NeuralVoice && voice is { Length: > 0 })
                {
                    // A voice that is not downloaded yet: the engine fetches it and
                    // announces itself when it is ready.
                    _config.NeuralVoiceName = voice;
                    _config.Save();
                }
                break;

            case "setVoiceEngine":
                SelectVoiceEngine(Text(message, "value"));
                break;

            case "setAvatar":
                SelectAvatar(Text(message, "value"));
                break;

            case "sight":
                // Whatever came back ends the wait, including a refusal — she should
                // answer without eyes rather than stand there for twenty seconds.
                var frame = Text(message, "image");
                if (Text(message, "error") is { Length: > 0 } trouble)
                    Log.Warn($"look failed in the face: {trouble}");

                // Described, never kept. A camera that opens and hands over a black
                // rectangle reports no error at all — this is the only place that
                // difference is visible. See Glance.
                if (!string.IsNullOrWhiteSpace(frame) && Sight.Inspect(frame) is { } glance)
                {
                    if (glance.LooksBlank) Log.Warn($"sight: {glance}");
                    else Log.Write($"sight: {glance}");
                }

                _looking?.TrySetResult(string.IsNullOrWhiteSpace(frame) ? null : frame);
                break;

            case "setMusic":
                SelectMusic(!message.TryGetProperty("value", out var m) || m.ValueKind != JsonValueKind.False);
                break;

            case "setMicrophone":
                SelectMicrophone(Text(message, "value") ?? "");
                break;

            case "setOutput":
                SelectOutput(Text(message, "value") ?? "");
                break;

            case "setCameraDevice":
                _config.CameraDevice = Text(message, "value") ?? "";
                _config.Save();
                Log.Write($"camera device: {Named(_config.CameraDevice)}");
                Announce();
                break;

            case "setWhisperCompute":
                SelectWhisperCompute(Text(message, "value") ?? "auto");
                break;

            case "setRoomHour":
                SelectRoomHour(message.TryGetProperty("value", out var h) && h.TryGetInt32(out var hour) ? hour : -1);
                break;

            case "selfTest":
                RunSelfTest().Forget("self-test");
                break;

            case "saveDiagnostics":
                SaveDiagnosticsAsync().Forget("saving diagnostics");
                break;

            case "openDataFolder":
                OpenDataFolder();
                break;
        }
    }

    private static string? Text(JsonElement message, string property) =>
        message.TryGetProperty(property, out var node) && node.ValueKind == JsonValueKind.String
            ? node.GetString()
            : null;

    /// Bumped only for a breaking change to the message contract; see PROTOCOL.md.
    private const int ProtocolVersion = 1;

    private void Announce() => _face.Send(new
    {
        type = "hello",
        protocol = ProtocolVersion,

        // What she is doing and wearing *now*. `state` and `emotion` are otherwise sent
        // only when they change, so a face attaching to a session already in progress
        // had no way to know either — and an expression can sit unchanged for a long
        // time, so a renderer that assumed neutral would simply be wrong until she next
        // felt something. Found by the renderer contract check.
        state = _state.ToString().ToLowerInvariant(),
        emotion = _mood.Name,
        emotionWeight = _mood.Weight,
        hasKey = _brain.IsReady || !_brain.NeedsApiKey,
        model = _brain.Description,
        profile = _config.Profile,
        voices = _voice.InstalledVoices().Select(name => new
        {
            value = name,
            label = _voice is NeuralVoice ? PiperStore.Pretty(name) : name.Replace("Microsoft ", "")
        }),
        voice = _voice.CurrentVoice,
        voiceEngine = _voice is NeuralVoice ? "neural" : "windows",
        ears = _ears?.EngineName ?? "not started",
        listening = _wantsToListen,
        avatar = AvatarUrl(),
        avatarFile = _config.AvatarFile ?? "",
        avatars = Avatars(),
        roomHour = _config.RoomHour,
        music = _config.Music,
        musicAvailable = _music.IsRunning,
        camera = _config.Camera,

        // Devices, so the drawer can offer a choice rather than inheriting whatever
        // Windows calls default — which on a machine with streaming software installed
        // is often a virtual endpoint that suits nothing. An empty value means "follow
        // the default", and is always the first option.
        microphones = AudioDevices.Capture().Select(d => new { value = d.Name, label = Label(d) }),
        microphone = _config.MicrophoneDevice ?? "",
        outputs = AudioDevices.Render().Select(d => new { value = d.Name, label = Label(d) }),
        output = _config.OutputDevice ?? "",
        cameraDevice = _config.CameraDevice ?? "",
        whisperCompute = _config.WhisperCompute ?? "auto",

        dev = _config.DevPanel ?? string.Equals(_config.Profile, "dev", StringComparison.OrdinalIgnoreCase)
    });

    private static string Label(AudioDevice device) =>
        device.IsDefault ? $"{device.Name} (Windows default)" : device.Name;

    /// The character file, on the read-only origin the host maps to her avatars folder.
    /// Null means the bust, which is the answer whenever anything is missing.
    private string? AvatarUrl()
    {
        var file = _config.AvatarFile;
        if (string.IsNullOrWhiteSpace(file)) return null;

        var path = Path.Combine(Paths.AvatarDir, file);
        if (File.Exists(path)) return $"https://octavia.avatar/{Uri.EscapeDataString(file)}";

        Log.Warn($"avatar '{file}' is not in {Paths.AvatarDir}; staying with the bust");
        return null;
    }

    /// What is actually in the folder, so choosing a character is a dropdown rather than
    /// a filename typed into a config file.
    private static IReadOnlyList<string> Avatars()
    {
        try
        {
            return Directory.EnumerateFiles(Paths.AvatarDir, "*.vrm")
                            .Select(Path.GetFileName)
                            .Where(name => name is not null)
                            .OrderBy(name => name, StringComparer.OrdinalIgnoreCase)
                            .ToList()!;
        }
        catch (Exception ex)
        {
            Log.Warn($"could not list {Paths.AvatarDir}: {ex.Message}");
            return [];
        }
    }

    /// An empty value means the bust. A name that is not there is refused rather than
    /// saved, so a bad selection cannot survive a restart.
    private void SelectAvatar(string? file)
    {
        file = file?.Trim() ?? "";

        if (file.Length > 0 && !File.Exists(Path.Combine(Paths.AvatarDir, file)))
        {
            Notice($"There is no '{file}' in her avatars folder.");
            Announce();
            return;
        }

        _config.AvatarFile = file;
        _config.Save();
        Log.Write($"avatar set to {(file.Length == 0 ? "the plaster bust" : file)}");
        Announce();
    }

    private void SelectRoomHour(int hour)
    {
        _config.RoomHour = hour is >= 0 and <= 23 ? hour : -1;
        _config.Save();
        Announce();
    }

    private void SaveKey(string? key)
    {
        if (string.IsNullOrWhiteSpace(key)) return;

        try
        {
            SecretStore.WriteApiKey(key);
            Notice("Key stored. It is sealed to this Windows account and never returns to the page.");
        }
        catch (Exception ex)
        {
            Log.Error($"key store failed: {ex.Message}");
            Notice("Could not store that key.");
        }

        Announce();
    }

    // ---- diagnostics -----------------------------------------

    /// Everything a report needs that only the running session knows.
    private HostSnapshot Snapshot() => new(
        Brain: _brain.Description,
        BrainReady: _brain.IsReady || !_brain.NeedsApiKey,
        Ears: _ears?.EngineName ?? "not started",
        Voice: _voice.EngineName,
        Listening: _wantsToListen,
        FaceBuilt: _faceBuilt,
        Faces: _face.Status,
        Music: MusicSummary());

    private string MusicSummary()
    {
        if (!_config.Music) return "off";
        if (!_music.IsRunning) return "no output device";

        var state = _music.State;
        return state.Playing
            ? $"{state.Bpm:0} bpm now (confidence {state.Confidence:0.00})"
            : "listening, nothing playing";
    }

    private async Task RunSelfTest()
    {
        _face.Send(new { type = "diagnostics", running = true });

        try
        {
            var snapshot = Snapshot();
            var checks = await SelfTest.RunAsync(_config, snapshot);

            foreach (var check in checks.Where(c => !c.Ok))
                Log.Warn($"self-test: {check.Name} — {check.Detail}");

            _face.Send(new
            {
                type = "diagnostics",
                running = false,
                checks,
                facts = SystemReport.Gather(_config, snapshot),
                log = Log.Tail(40)
            });
        }
        catch (Exception ex)
        {
            Log.Error("self-test failed", ex);
            _face.Send(new { type = "diagnostics", running = false });
            Notice("The self-test itself failed. The log has the detail.");
        }
    }

    /// The button that matters: a single zip, saved wherever the person using her wants
    /// it, that can be sent back from a machine nobody here can reach.
    internal async Task SaveDiagnosticsAsync()
    {
        var dispatcher = System.Windows.Application.Current?.Dispatcher;
        if (dispatcher is null)
        {
            Log.Warn("no dispatcher, so the diagnostics dialog cannot be shown");
            return;
        }

        // The request arrives on a socket thread. A file dialog is a UI object, so it is
        // both built and shown on the UI thread — constructing it out here throws, and
        // because this method is fire-and-forget the throw is silent.
        var chosen = await dispatcher.InvokeAsync(() =>
        {
            var dialog = new Microsoft.Win32.SaveFileDialog
            {
                Title = "Save Octavia diagnostics",
                FileName = DiagnosticsBundle.SuggestedFileName(),
                DefaultExt = ".zip",
                Filter = "Zip archive (*.zip)|*.zip",
                AddExtension = true
            };

            return dialog.ShowDialog() == true ? dialog.FileName : null;
        });

        if (chosen is null) return;

        try
        {
            await Task.Run(() => DiagnosticsBundle.WriteAsync(chosen, _config, Snapshot()));
            _face.Send(new { type = "diagnosticsSaved", path = chosen });
            Notice("Diagnostics saved. Open it and read README.txt before sending it on.");
        }
        catch (Exception ex)
        {
            Log.Error("diagnostics bundle failed", ex);
            Notice($"Could not write the diagnostics file: {ex.Message}");
        }
    }

    private void OpenDataFolder()
    {
        try
        {
            System.Diagnostics.Process.Start("explorer.exe", Paths.DataDir);
        }
        catch (Exception ex)
        {
            Log.Warn($"could not open the data folder: {ex.Message}");
        }
    }

    // ---- listening -------------------------------------------

    public void ToggleListening()
    {
        if (_wantsToListen) StopListening();
        else StartListening();
    }

    private void StartListening()
    {
        _wantsToListen = true;

        if (_ears is not null)
        {
            EngageEars();
            return;
        }

        if (_earsOpening) return;
        _earsOpening = true;

        // Whisper's first listen may download the model; never block the message loop.
        _ = Task.Run(async () =>
        {
            try
            {
                var ears = await OpenEarsAsync();
                ears.Recognized += OnHeard;
                ears.Trouble += Notice;
                ears.Hypothesised += partial =>
                    _face.Send(new { type = "caption", who = "You", text = partial, tentative = true });
                _ears = ears;
                Log.Write($"ears open: {ears.EngineName}");
                if (_wantsToListen) EngageEars();
            }
            catch (Exception ex)
            {
                Log.Error("cannot open ears", ex);
                Notice(Explain(ex));
                _wantsToListen = false;
                if (_state is AgentState.Listening) SetState(AgentState.Idle);
            }
            finally
            {
                _earsOpening = false;
                Announce();
            }
        });
    }

    private async Task<ISpeechRecognizer> OpenEarsAsync()
    {
        if (!string.Equals(_config.Recognizer, "whisper", StringComparison.OrdinalIgnoreCase))
            return new SystemSpeechRecognizer(_config.RecognitionCulture);

        try
        {
            var modelPath = await WhisperModelStore.EnsureAsync(_config.WhisperModel, Notice);
            return new WhisperRecognizer(
                modelPath, _config.WhisperModel, _config.WhisperLanguage,
                _config.WhisperCompute, _config.WhisperThreads, _config.MicrophoneDevice);
        }
        catch (Exception ex)
        {
            Log.Warn($"whisper unavailable, falling back to Windows recognizer: {ex.Message}");
            Notice("Whisper could not start; using the Windows recognizer for now.");
            return new SystemSpeechRecognizer(_config.RecognitionCulture);
        }
    }

    private void EngageEars()
    {
        _ears?.Start();
        _meter.Start();
        if (_state is AgentState.Idle) SetState(AgentState.Listening);
        Announce();
    }

    private void StopListening()
    {
        _wantsToListen = false;
        _ears?.Stop();
        _meter.Stop();
        if (_state is AgentState.Listening) SetState(AgentState.Idle);
        Announce();
    }

    private void OnHeard(string text, float confidence)
    {
        if (_responding) return;
        if (text.Length < _config.MinUtteranceChars) return;

        if (confidence < _config.MinConfidence)
        {
            Log.Debug($"discarded (confidence {confidence:0.00}): {text}");
            return;
        }

        Consider(text).Forget("weighing something overheard");
    }

    /// Everything the ears produce comes through here; everything *typed* does not.
    /// If you took the trouble to type it, you meant it, and a gate that second-guesses
    /// a deliberate act is only ever wrong.
    private async Task Consider(string text)
    {
        var verdict = await _gate.JudgeAsync(text);

        if (verdict.Answer)
        {
            await RespondTo(text);
            return;
        }

        // Never silently. "She ignored me" has to be a question with an answer, and the
        // face shows it faintly so the reason is visible without opening the log.
        Log.Write($"overheard ({verdict.Why}, {verdict.Cost.TotalMilliseconds:0} ms): {text}");
        _face.Send(new { type = "overheard", text, why = verdict.Why });
    }

    // ---- eyes ------------------------------------------------

    /// Set while a `look` is outstanding. One at a time: a second request would leave
    /// the first waiting forever for a frame that went to the wrong place.
    private TaskCompletionSource<string?>? _looking;

    /// A still, but only if the question genuinely needs one and she is allowed.
    ///
    /// Three gates before a camera opens, and all three are cheap and legible: the
    /// setting is on, the words need eyes, and the brain has any. Nothing here consults
    /// a model — the decision to open a camera in someone's home has to be auditable by
    /// reading it. See `Sight`.
    private async Task<string?> MaybeLookAsync(string userText, CancellationToken cancel)
    {
        if (!_config.Camera || !Sight.WantsEyes(userText)) return null;

        // A brain with no eyes would only be told there was a picture it cannot see.
        if (_brain is not ClaudeBrain)
        {
            Notice("She would need her Claude brain to see anything.");
            return null;
        }

        if (_looking is not null) return null;

        var waiting = new TaskCompletionSource<string?>(TaskCreationOptions.RunContinuationsAsynchronously);
        _looking = waiting;

        try
        {
            _face.Send(new { type = "look" });

            // The face has to raise a permission prompt on first use, and a person has
            // to answer it. Long enough for that, and then she gets on without it.
            using var timeout = CancellationTokenSource.CreateLinkedTokenSource(cancel);
            timeout.CancelAfter(TimeSpan.FromSeconds(20));

            await using (timeout.Token.Register(() => waiting.TrySetResult(null)))
            {
                var image = await waiting.Task;
                if (image is null) Log.Warn("look: no frame came back");
                else Log.Write($"look: got a frame, {image.Length * 3 / 4 / 1024} KB");
                return image;
            }
        }
        finally
        {
            _looking = null;
        }
    }

    // ---- a turn ----------------------------------------------

    private async Task RespondTo(string userText)
    {
        if (_responding) return;

        if (!_brain.IsReady && _brain.NeedsApiKey)
        {
            Notice("No API key yet. Paste one below and try again.");
            _face.Send(new { type = "needKey" });
            return;
        }

        _responding = true;
        _turn?.Cancel();
        _turn = new CancellationTokenSource();
        var cancel = _turn.Token;

        _ears?.Mute();
        _meter.Stop();

        _face.Send(new { type = "caption", who = "You", text = userText });
        _face.Send(new { type = "turn", who = "you", text = userText });
        SetState(AgentState.Thinking);

        var reply = new StringBuilder();

        try
        {
            var now = new Situation(
                Persona.Music(_music.State.Playing, _music.State.Bpm),
                await MaybeLookAsync(userText, cancel));

            await foreach (var sentence in _brain.RespondAsync(userText, now, cancel))
            {
                if (cancel.IsCancellationRequested) break;

                reply.Append(reply.Length > 0 ? " " : "").Append(sentence);
                _face.Send(new { type = "caption", who = "Octavia", text = reply.ToString() });
                Feel(Moods.Read(sentence));
                _voice.Say(sentence);
            }

            if (reply.Length > 0)
                _face.Send(new { type = "turn", who = "octavia", text = reply.ToString() });
            else if (!cancel.IsCancellationRequested)
                Notice("She had nothing to say.");
        }
        catch (OperationCanceledException)
        {
            // hushed mid-thought
        }
        catch (Exception ex)
        {
            Log.Error("turn failed", ex);
            Notice(Explain(ex));
            _face.Send(new { type = "caption", who = "", text = "Something went wrong reaching the model." });
        }
        finally
        {
            _responding = false;
            _gate.Answered();
            if (!_voice.IsSpeaking) OnVoiceFinished();
        }
    }

    private static string Explain(Exception ex) => ex switch
    {
        InvalidOperationException => ex.Message,
        HttpRequestException => "No route to the API. Check the network.",
        _ => ex.Message.Length > 140 ? ex.Message[..140] : ex.Message
    };

    private void Hush()
    {
        _turn?.Cancel();
        _voice.Hush();
    }

    private void OnVoiceFinished()
    {
        if (_responding || _voice.IsSpeaking) return;

        // The turn is over; her face should not keep the last sentence's mood on it.
        Feel(Mood.Neutral);

        if (_wantsToListen)
        {
            _ears?.Unmute();
            _meter.Start();
            SetState(AgentState.Listening);
        }
        else
        {
            SetState(AgentState.Idle);
        }
    }

    /// Sent only when it changes: an expression is a change of face, and repeating one
    /// every sentence would keep restarting the movement towards it.
    private void Feel(Mood mood)
    {
        if (mood.Name == _mood.Name && Math.Abs(mood.Weight - _mood.Weight) < 0.01) return;
        _mood = mood;
        _face.Send(new { type = "emotion", value = mood.Name, weight = mood.Weight });
    }

    private void SetState(AgentState state)
    {
        if (_state == state) return;
        _state = state;

        // Everything she says comes back through the loopback a few milliseconds later.
        // Holding the analysis while she talks is what stops her hearing herself and
        // deciding it is music; the tempo she already had keeps running underneath.
        _music.Hold = state is AgentState.Speaking;

        _face.Send(new { type = "state", value = state.ToString().ToLowerInvariant() });
    }

    /// Notices are the things she thought were worth interrupting for, so they belong in
    /// the log as well as on her face — by the time a bundle arrives, the one that
    /// mattered has long since faded off the screen.
    private void Notice(string text)
    {
        Log.Write($"notice: {text}");
        _face.Send(new { type = "notice", text });
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _turn?.Cancel();
        _turn?.Dispose();
        _ears?.Dispose();
        _meter.Dispose();
        _music.Dispose();
        _gate.Dispose();
        _voice.Dispose();
        _brain.Dispose();
    }
}
```
