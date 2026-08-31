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

    /// A second analyser fed from the microphone, so a speaker in the room is not silence
    /// to her. Only started when MusicFromRoom is on and her ears are open.
    private readonly MusicWatcher _roomMusic = new() { Name = "room" };

    /// **One** local microphone, shared by the ears and the room-music analyser.
    ///
    /// Owned here rather than inside the recogniser precisely so the two can be separated:
    /// a face taking the floor moves what she *transcribes* without moving what she hears
    /// around her. Opening the device twice would be the obvious way to get this wrong.
    private LocalMicSource? _localMic;
    private readonly PcmFramer _roomFramer = new();
    private readonly AttentionGate _gate;
    private readonly Brain.Tools.ToolRegistry _tools;

    private ISpeechRecognizer? _ears;
    private bool _earsOpening;
    private CancellationTokenSource? _turn;
    private AgentState _state = AgentState.Idle;
    private bool _responding;
    private bool _wantsToListen;
    private bool _faceBuilt;
    private bool _faceSpoke;

    /// Whether the face has ever said `ready`. A renderer whose JavaScript failed to parse
    /// never runs the code that reports errors, so silence is the only symptom the host can
    /// observe — and silence is what nothing was watching for. See ROADMAP.md stage 10a.
    internal bool FaceSpoke => _faceSpoke;
    private Mood _mood = Mood.Neutral;
    private float _lastSentLevel = -1f;
    private bool _disposed;

    public OctaviaSession(OctaviaConfig config, IFaceTransport face)
    {
        _config = config;
        _face = face;
        _gate = new AttentionGate(config);
        _tools = new Brain.Tools.ToolRegistry(config);
        _meter.Device = config.MicrophoneDevice;
        _music.Device = config.OutputDevice;

        // Started in the background: an MCP server that is slow to come up must not
        // delay her being able to talk, and one that never comes up must not stop it.
        if (config.McpServers.Count > 0)
            StartTools().Forget("starting the tool servers");
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

        // Only the face holding the floor is listened to. Frames from anyone else are
        // dropped rather than mixed — two rooms transcribed into one sentence is worse
        // than one of them being ignored.
        _face.AudioReceived += (from, pcm) =>
        {
            if (_floor == from) _faceMic?.Push(pcm, pcm.Length);
        };

        // A phone that walks out of range must not hold her ears for ever.
        _face.FaceDeparted += from => { if (_floor == from) ReleaseFloor("disconnected and so released"); };

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

    // ---- her hands --------------------------------------------

    /// Connects the configured MCP servers and reports what they offer.
    ///
    /// **She cannot yet call these.** The seam, the client, the risk policy and the
    /// confirmation rule are done and tested; the brain-side tool loop is not, and it is
    /// not written blind — it changes the main conversation path and there is no API key
    /// on this machine to verify it against. What this gives today is a configured,
    /// visible, reachable integration, which is what the next step needs to exist.
    /// See ROADMAP.md stage 12.
    private async Task StartTools()
    {
        try
        {
            await _tools.StartAsync();
            var offered = await _tools.ListAsync();

            if (offered.Count > 0)
                Log.Write($"tools available: {string.Join(", ", offered.Select(t => t.Name))}");

            Announce();
        }
        catch (Exception ex)
        {
            Log.Error("the tool servers would not start", ex);
        }
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

        /* The shared source is torn down too, or the ears would reopen onto the old device:
           a capture device is chosen when the stream opens and not afterwards, and this one
           outlives the recogniser precisely so the room-music analyser can keep it. */
        _localMic?.Dispose();
        _localMic = null;
        _roomFramer.Frame -= _roomMusic.Push;

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

        // Her voice, to any face that asked for it. Opt-in, so this costs nothing at all
        // until somebody wants to hear her in another room — see `Face.Want`.
        voice.Audio += pcm => _face.SendAudio(pcm);
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

    private void OnFaceMessage(FaceMessage inbound)
    {
        var message = inbound.Body;
        if (!message.TryGetProperty("type", out var typeNode)) return;

        var kind = typeNode.GetString();

        // A person did this, through this face. Everything else — `ready`, `sight`,
        // settings echoes — is a renderer talking, not somebody in a room.
        if (kind is "say" or "listen" or "hush") _lastSpokenThrough = inbound.From;

        switch (kind)
        {
            case "ready":
                _faceSpoke = true;
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
                // A frame from a face that was never asked is dropped, and said out loud.
                // Before faces had identity this could not be detected at all: `look` went
                // to everyone and the first answer back won, arbitrarily. A face answering
                // a `look` it never received is a bug worth seeing, not one to swallow.
                if (_looking is not { } pending)
                {
                    Log.Warn($"sight from face {inbound.From} with no look outstanding; ignored");
                    break;
                }

                if (inbound.From != pending.Face)
                {
                    Log.Warn($"sight from face {inbound.From}, but {pending.Face} was asked; ignored");
                    break;
                }

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

                pending.Waiting.TrySetResult(string.IsNullOrWhiteSpace(frame) ? null : frame);
                break;

            case "setStats":
                _config.ShowStats = !message.TryGetProperty("value", out var s) || s.ValueKind != JsonValueKind.False;
                _config.Save();
                Announce();
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

            // The one sense that is off by default, and the only one that had no switch:
            // the setting existed and the protocol carried the device, but nothing in the
            // face could turn it on. Logged at warn on the way up, because a camera coming
            // on in someone's home should leave a mark that is easy to find later.
            // Push-to-talk. Deliberately not `listen`, which toggles *her own* microphone
            // — a different thing that has to keep working independently.
            case "talking":
                OnTalking(inbound.From,
                    !message.TryGetProperty("value", out var talk) || talk.ValueKind != JsonValueKind.False);
                break;

            case "setCamera":
                var seeing = !message.TryGetProperty("value", out var cam) || cam.ValueKind != JsonValueKind.False;
                _config.Camera = seeing;
                _config.Save();

                if (seeing) Log.Warn("camera enabled from settings");
                else Log.Write("camera disabled from settings");

                Announce();
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
        stats = _config.ShowStats,

        // What a face has to know to play her voice, announced rather than assumed. The
        // rate comes from the live voice's own model config, so it changes with the voice
        // — and `hello` is re-sent on every settings change, which carries the new rate
        // with it. A face must re-read this each time rather than caching it once.
        // Whether the host will take audio *from* a face at all, so a client does not offer
        // a microphone button that could only fail — the same courtesy `camera` already does.
        micAccepted = _ears is WhisperRecognizer,

        audioAvailable = _voice.AudioFormat is not null,
        audioRate = _voice.AudioFormat?.Rate ?? 0,
        audioBits = _voice.AudioFormat?.Bits ?? 0,
        audioChannels = _voice.AudioFormat?.Channels ?? 0,

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

        // What she has been given hands for. Reported even though she cannot call them
        // yet, because "is the integration actually connected" is the first question
        // anyone will ask and it should not need a log to answer.
        toolServers = _tools.Providers.Select(p => new { name = p.Name, ready = p.IsReady }),

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

        /* The device is named even when she *is* hearing something, because "she does
           not dance" is almost never the analyser and almost always the music going to
           a different endpoint than the one she tapped. Loopback hears one output; a
           machine has four. Saying which turns a mystery into a dropdown. */
        var state = _music.State;
        return state.Playing
            ? $"{state.Bpm:0} bpm now (confidence {state.Confidence:0.00}) on '{_music.DeviceName}'"
            : $"listening to '{_music.DeviceName}', nothing playing through it";
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

                /* One device, two listeners. The recogniser is handed the source rather
                   than building its own, so the room-music analyser below can frame the
                   same microphone independently — and so a face taking the floor moves
                   only what she transcribes. */
                if (ears is WhisperRecognizer recogniser)
                {
                    _localMic ??= new LocalMicSource(_config.MicrophoneDevice);
                    recogniser.UseSource(_localMic);
                }

                /* Room music. The microphone is already open and these are the very same
                   frames the voice detector is reading, so hearing a speaker across the
                   room costs one subscription and no extra capture. See ROADMAP.md 11a. */
                if (_config.MusicFromRoom && ears is WhisperRecognizer whisper)
                {
                    /* The loopback wins when it has something.
                       Both sources write to the same face state, so without a rule they
                       fight: the tempo flickers between whatever this machine is playing
                       and whatever is in the room, and neither reading is trustworthy.
                       Loopback is the better witness — clean dynamics, no room, no gain
                       control — so the microphone only speaks when it is silent. */
                    _roomMusic.Changed += state =>
                    {
                        if (_music.State.Playing) return;
                        _face.Send(new
                        {
                            type = "music",
                            playing = state.Playing,
                            bpm = state.Bpm,
                            energy = state.Energy,
                            beat = false
                        });
                    };
                    _roomMusic.Beat += () =>
                    {
                        if (_music.State.Playing) return;
                        _face.Send(new { type = "music", beat = true });
                    };

                    /* Bound to the **local microphone**, not to whatever the recogniser is
                       currently consuming.

                       This used to be `whisper.Audio += _roomMusic.Push`, which was exactly
                       right while there was only one microphone in the world. Once a face
                       can hand the ears a phone, that subscription would quietly follow it
                       — and she would report the tablet's kitchen radio as the music around
                       *her*. Everything would appear to work.

                       So the local source keeps running and is framed separately for music
                       even while a face holds the floor for speech. Speech moves rooms; her
                       sense of what is playing here does not. The extra work is one
                       byte-to-float loop over 16 kHz mono, which is nothing. */
                    _roomMusic.StartFromFrames(SileroVad.SampleRate);
                    _localMic ??= new LocalMicSource(_config.MicrophoneDevice);
                    _roomFramer.Frame += _roomMusic.Push;
                    _localMic.Data += _roomFramer.Push;
                    _localMic.Start();

                    Log.Write("music: also listening for a beat in this room, through the local microphone");
                }
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

    // ---- the floor -------------------------------------------

    /* Push-to-talk from a face, and it is load-bearing rather than a convenience.
       A held button has already answered "was that addressed to me?", so the attention
       gate does not apply to this path and item 7 is not needed for it. One talker at a
       time also means one Whisper, so the earlier worry about sizing two concurrent
       transcriptions against eight cores does not arise. */

    /// Which face is holding the floor, if any. One at a time, first press wins.
    private FaceId? _floor;
    private FaceAudioSource? _faceMic;
    private System.Threading.Timer? _floorTimeout;

    /// A phone in a pocket with a stuck button must not own her ears indefinitely.
    private static readonly TimeSpan FloorLimit = TimeSpan.FromSeconds(60);

    private void OnTalking(FaceId from, bool holding)
    {
        if (holding) TakeFloor(from);
        else if (_floor == from) ReleaseFloor("released");
    }

    private void TakeFloor(FaceId from)
    {
        if (_floor is { } held)
        {
            if (held == from) return;

            // Refused out loud rather than in silence. Two half-sentences interleaved into
            // one transcript is worse than being told to wait.
            _face.Send(new { type = "notice", text = "Someone else is talking to her." }, from);
            Log.Write($"face {from} pressed while {held} holds the floor; refused");
            return;
        }

        // Talking over her is an interrupt, not something to record on top of. `hush`
        // already does exactly the right thing.
        if (_state is AgentState.Speaking or AgentState.Thinking) Hush();

        if (_ears is not WhisperRecognizer recogniser)
        {
            _face.Send(new { type = "notice", text = "Her ears are not open yet." }, from);
            return;
        }

        _floor = from;
        _faceMic = new FaceAudioSource($"a face ({from})");

        // The local microphone is muted for *speech* — she must not transcribe a blend of
        // this room and the other one. It keeps running for the room-music analyser, which
        // is a different question entirely. See the note on _localMic.
        recogniser.UseSource(_faceMic);
        _faceMic.Start();

        _floorTimeout = new System.Threading.Timer(
            _ => ReleaseFloor("timed out"), null, FloorLimit, Timeout.InfiniteTimeSpan);

        Log.Write($"face {from} has the floor");
    }

    private void ReleaseFloor(string why)
    {
        if (_floor is not { } held) return;

        _floor = null;
        _floorTimeout?.Dispose();
        _floorTimeout = null;

        /* Ending the utterance here rather than waiting for the voice detector to guess.
           A released button is a far better end-of-sentence marker than silence is, and a
           face that vanished mid-stream means the same thing — otherwise the floor is held
           for ever by something that is no longer there. */
        if (_ears is WhisperRecognizer recogniser)
        {
            recogniser.Flush();
            if (_localMic is not null) recogniser.UseSource(_localMic);
        }

        _faceMic?.Dispose();
        _faceMic = null;

        Log.Write($"face {held} {why} the floor");
    }

    // ---- eyes ------------------------------------------------

    /// Set while a `look` is outstanding, **with the face it was sent to**. One at a
    /// time: a second request would leave the first waiting forever for a frame that
    /// went to the wrong place.
    ///
    /// The face matters as much as the promise. `look` used to be broadcast, so with a
    /// tablet and the desktop both attached, one question opened *both* cameras and lit
    /// the privacy marker in an empty room — breaking the promise `camera.js` makes in
    /// its own header: "it is never opened unasked".
    private (FaceId Face, TaskCompletionSource<string?> Waiting)? _looking;

    /// The last face a *person* spoke through — `say`, `listen`, `hush`. Where `look` goes.
    ///
    /// **This is not turn ownership** and must not grow into it; that is Stage 14 item 5.
    /// It is one field that answers one question well enough to fix the camera bug:
    /// whoever last asked her something is the one holding a device worth looking through.
    /// Null means nobody has, which is the case when the utterance arrived through the
    /// PC's own microphone — so it falls back to the built-in page.
    private FaceId? _lastSpokenThrough;

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

        // One face, not all of them. Falls back to the built-in page when nobody has
        // spoken through a remote face — an utterance from the PC's own microphone.
        var target = _lastSpokenThrough ?? _face.BuiltInFace;
        if (target is null)
        {
            Log.Warn("look: no face to ask, so she answers without eyes");
            return null;
        }

        var waiting = new TaskCompletionSource<string?>(TaskCreationOptions.RunContinuationsAsynchronously);
        _looking = (target.Value, waiting);

        try
        {
            Log.Write($"look: asking face {target.Value}");
            _face.Send(new { type = "look" }, target);

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

        // Each provider owns a child process, so this is a real shutdown rather than
        // bookkeeping: skipping it leaves an MCP server running after she closes.
        // Blocking here is deliberate — Dispose has no async form to hand this to, and
        // an orphaned process is worse than a two-second wait on the way out.
        _floorTimeout?.Dispose();
        _faceMic?.Dispose();
        _localMic?.Dispose();
        _roomMusic.Dispose();
        _tools.DisposeAsync().AsTask().GetAwaiter().GetResult();
    }
}
```
