---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\OctaviaSession.cs
---

# src\Octavia.Core\OctaviaSession.cs

```csharp
using System.Collections.Concurrent;
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
    private readonly Brain.Tools.ToolRegistry _tools;

    private ISpeechRecognizer? _ears;
    private CancellationTokenSource? _turn;
    private bool _responding;
    private bool _wantsToListen;
    private bool _faceBuilt;
    private bool _faceSpoke;

    /// Whether the face has ever said `ready`. A renderer whose JavaScript failed to parse
    /// never runs the code that reports errors, so silence is the only symptom the host can
    /// observe — and silence is what nothing was watching for. See ROADMAP.md stage 10a.
    internal bool FaceSpoke => _faceSpoke;
    private float _lastSentLevel = -1f;
    private bool _disposed;

    // ---- rooms -----------------------------------------------

    /* One being, N rooms. See Room, and Stage 14 item 9.

       Two faults were being fixed at once and they are worth keeping apart. One is that no
       `set*` case looked at where a message came from, so a phone at the gym could open the
       microphone in an empty house. The other is that there was one conversation and every
       face was a window onto it — typing at her on a handset put the words on the desk and
       played the answer in a room nobody was in. */

    /// Where each face is. A face declares its room in `ready`; saying nothing means the
    /// host, which is why no existing renderer changes behaviour.
    private readonly ConcurrentDictionary<FaceId, RoomId> _where = new();

    /// What each face says it can do — `["mic", "camera"]`. A face that declares nothing is
    /// a renderer, which is a legal face and always has been.
    private readonly ConcurrentDictionary<FaceId, string[]> _senses = new();

    private readonly ConcurrentDictionary<RoomId, Room> _rooms = new();

    /// Who is in each room, as an array replaced wholesale on every membership change.
    ///
    /// The same instinct as `Face.Skip` on the socket server, for the same reason: her voice
    /// thread reads this forty times a second while a phone can join from a socket thread,
    /// and enumerating a dictionary through a mutation throws. Swapping a reference is
    /// atomic; editing a map is not.
    private volatile IReadOnlyDictionary<RoomId, FaceId[]> _members =
        new Dictionary<RoomId, FaceId[]>();

    private readonly object _rooming = new();

    /// The room she is attending. **One at a time, deliberately** — she has one voice, one
    /// Whisper, one `_responding` flag and one cancellation source, and a being that holds
    /// two conversations at once is a worse simulation rather than a better one. The other
    /// room is refused out loud; see `RespondTo`.
    private RoomId _attending = RoomId.Host;

    public OctaviaSession(OctaviaConfig config, IFaceTransport face)
    {
        _config = config;
        _face = face;
        _tools = new Brain.Tools.ToolRegistry(config);
        _meter.Device = config.MicrophoneDevice;
        _music.Device = config.OutputDevice;

        // Started in the background: an MCP server that is slow to come up must not
        // delay her being able to talk, and one that never comes up must not stop it.
        if (config.McpServers.Count > 0)
            StartTools().Forget("starting the tool servers");
        _brain = string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase)
            ? new LocalBrain(config, _tools)
            : new ClaudeBrain(config, _tools);
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
            // Either claim will do: a held button, or a room that has been left listening.
            // Still exactly one face at a time — two rooms mixed into one sentence is worse
            // than one of them being ignored.
            if (_floor == from || (_floor is null && _openFace == from))
                _faceMic?.Push(pcm, pcm.Length);
        };

        // A phone that walks out of range must not hold her ears for ever, and must not
        // leave a room holding an address nothing answers on.
        _face.FaceDeparted += from =>
        {
            if (_floor == from) ReleaseFloor("disconnected and so released");

            // A handset that walked out of range is not a room that is still listening, and
            // leaving it open would hold her ears for a device that is gone.
            if (_openFace == from) CloseRoomListening("the face left");

            Departed(from);
        };

        /* Both of these are the **host machine's** microphone and the host machine's output
           mix, so they go to the host room and nowhere else.

           `music` is the one the spec calls out by name, and it is the room-microphone trap
           from Stage 14 item 2 wearing different clothes: a tempo measured from the speakers
           under this desk means nothing at all to somebody holding a phone in a gym. */
        _meter.LevelChanged += level =>
        {
            if (Math.Abs(level - _lastSentLevel) < 0.02f) return;
            _lastSentLevel = level;
            ToRoom(RoomId.Host, new { type = "level", value = level });
        };

        _music.Changed += state => ToRoom(RoomId.Host, new
        {
            type = "music",
            playing = state.Playing,
            bpm = state.Bpm,
            energy = state.Energy,
            beat = false
        });

        // The beat is sent on its own and never coalesced with the state above: a beat
        // that arrives 200 ms late is worse than one that never arrived at all.
        _music.Beat += () => ToRoom(RoomId.Host, new { type = "music", beat = true });

        if (_config.Music) StartMusic().Forget("starting to listen for music");
    }

    // ---- who is where -----------------------------------------

    /// The room, made if this is the first face to name it. Rooms live in memory only, like
    /// the conversation they hold.
    private Room RoomNamed(RoomId id) => _rooms.GetOrAdd(id, key => new Room(key, _config));

    /// The room this machine is standing in. Shorthand, because a great deal belongs to it:
    /// its microphone, its speakers, its settings, and the built-in page.
    private Room Host => RoomNamed(RoomId.Host);

    /// Whose words the recogniser is currently turning into text. The floor-holder's room
    /// while a face holds it, this machine's otherwise — the ears follow the microphone,
    /// not the machine the engine happens to run on.
    ///
    /// **Live, and therefore only right while somebody is still speaking.** A finished
    /// utterance is attributed with `_owed` instead; see the note there.
    ///
    /// A press wins over an open room: the floor is the louder claim, and it is what a person
    /// just did rather than what they switched on earlier.
    private RoomId EarsRoom =>
        _floor is { } holder ? RoomOf(holder).Id
        : _openFace is { } streaming ? RoomOf(streaming).Id
        : RoomId.Host;

    /// <summary>
    /// The utterance that has been flushed but not yet transcribed, and who it belongs to.
    /// </summary>
    ///
    /// <remarks>
    /// **Transcription outlives the press, and nothing used to carry the speaker across the
    /// gap.** `talking:false` releases the floor synchronously — `_floor` becomes null — and
    /// only then does `Flush` hand the samples to Whisper, which answers on its own thread
    /// some time later. By then `EarsRoom` reads the empty floor and says `host`, so a
    /// sentence spoken into a phone was judged by the *desktop's* attention gate and, if it
    /// survived that, answered in the desktop's room. `_attending` was already set correctly
    /// by `TakeFloor` and `RespondTo` promptly overwrote it with the wrong one.
    ///
    /// It is latched here at flush time instead, and read exactly once by whoever the words
    /// turn out to belong to.
    ///
    /// `Deliberate` rides along because the same gap hides the other half: **a held button
    /// has already said "this is for you"**, so the gate must not second-guess it. The code
    /// claimed push-to-talk bypassed the gate; it did not, and anything under twelve
    /// characters was dropped as *"too short to be addressed to anyone"* — press, say "yes",
    /// get ignored. `RoomChecks` runs with `Gate = "off"`, which is why the one harness that
    /// drives a face taking the floor could never see it.
    /// </remarks>
    private (RoomId Room, bool Deliberate)? _owed;

    /// <summary>The face streaming its microphone continuously, and the room it is in.</summary>
    ///
    /// <remarks>
    /// **Stage 14 item 6.** A room asking to `listen` is not asking to open the microphone on
    /// the machine she runs on — it is saying *"transcribe what I am sending"*. That is the
    /// same message meaning two different things in two places, which is what item 3 of Stage
    /// 15 concluded it should mean everywhere once the server holds no device at all.
    ///
    /// **Deliberately not the floor.** `_floor` is a single holder with a sixty-second limit,
    /// and a face that simply held it forever would starve the desk and then time out. This is
    /// a separate, quieter claim: the room is *open*, and a press can still take the floor on
    /// top of it and hand it back afterwards.
    ///
    /// One at a time, because the recogniser has one source and one voice — the same reason
    /// rooms are serialised everywhere else.
    /// </remarks>
    private FaceId? _openFace;
    private RoomId _openRoom = RoomId.Host;

    private Room RoomOf(FaceId face) =>
        RoomNamed(_where.TryGetValue(face, out var id) ? id : RoomId.Host);

    /// Lock-free, and it has to be: her voice reads this on the sound card's thread while a
    /// phone can join from a socket thread. See `_members`.
    private FaceId[] FacesIn(RoomId room) =>
        _members.TryGetValue(room, out var faces) ? faces : [];

    private void ToRoom(RoomId room, object message) => _face.SendMany(message, FacesIn(room));

    /// A face is known from the first thing it ever says, in the host room until `ready`
    /// puts it somewhere else. Waiting for `ready` would leave `attach-face.ps1` and the
    /// checks — which connect and speak without one — addressable by nothing.
    private void Seen(FaceId face)
    {
        if (_where.ContainsKey(face)) return;
        Join(face, RoomId.Host);
    }

    private void Join(FaceId face, RoomId room)
    {
        var moved = !_where.TryGetValue(face, out var was) || was != room;
        _where[face] = room;
        RoomNamed(room);

        if (moved) Remember();
        if (moved && room != RoomId.Host) Log.Write($"face {face} is in room '{room}'");
    }

    private void Departed(FaceId face)
    {
        if (!_where.TryRemove(face, out _)) return;
        _senses.TryRemove(face, out _);
        Remember();
    }

    /// Rebuilds the membership snapshot wholesale under the lock, and publishes it with one
    /// reference swap. Nothing reads the map while it is being edited, because it never is.
    private void Remember()
    {
        lock (_rooming)
        {
            var built = new Dictionary<RoomId, List<FaceId>>();
            foreach (var (face, room) in _where)
            {
                if (!built.TryGetValue(room, out var list)) built[room] = list = [];
                list.Add(face);
            }

            _members = built.ToDictionary(pair => pair.Key, pair => pair.Value.ToArray());
        }
    }

    /// Every face she could speak to, for the per-face `hello`.
    private IEnumerable<FaceId> KnownFaces() => _where.Keys;

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
        Notice(Host, $"Whisper will use {wanted} the next time she starts.");
        Announce();
    }

    private void Listen(IVoice voice)
    {
        // Her mouth moves where she is talking. A desk avatar mouthing a conversation
        // happening on a handset in another building is precisely the thing rooms exist to
        // stop — and it is the same buffer, so the two rooms cannot disagree.
        voice.Viseme += (openness, shape) =>
            ToRoom(_attending, new { type = "viseme", value = openness, shape });

        voice.Started += () => SetState(RoomNamed(_attending), AgentState.Speaking);
        voice.Finished += OnVoiceFinished;

        // A voice engine that would not start is about *her*, not about a room, so every
        // face is told: they are all wearing the result.
        voice.Trouble += Notice;

        // Her voice, to the faces in the room she is attending that asked for it. Opt-in,
        // so this costs nothing at all until somebody wants to hear her — see `Face.Want`.
        voice.Audio += pcm => _face.SendAudio(pcm, FacesIn(_attending));
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

    /* The authority table. Every face→host message is one of three things, and the check is
       on the *room the message came from*, here, before the switch acts.

       Hiding a button in the renderer is not the fix and never was: a face that can send
       `listen` by hand can still open the microphone in an empty house. `hello.controls` is
       the hint for the interface; this is the enforcement, and both are needed — without the
       guard a remote face drives the hardware anyway, and without the hint a phone shows a
       microphone button that silently does nothing, which is its own kind of broken. */

    /// Acts on the machine she runs on. Only from a face in the host room.
    private static readonly HashSet<string> HostOnly =
    [
        "listen", "setMicrophone", "setOutput", "setMusic",
        "setWhisperCompute", "openDataFolder", "saveDiagnostics"
    ];

    /// Changes *her*, not a room, so any room may — and every room is told, because every
    /// face is now wearing it.
    private static readonly HashSet<string> BeingWide =
    [
        "setKey", "setVoice", "setVoiceEngine", "setAvatar", "setRoomHour", "setStats"
    ];

    /// One line per room per kind of refusal. A wall tablet with a stuck button would
    /// otherwise write the same sentence into her log until the disk filled.
    private readonly HashSet<string> _refused = [];

    private bool Refuse(FaceId face, RoomId room, string kind)
    {
        _face.Send(new { type = "notice", text = "That belongs to the machine she runs on." }, face);

        lock (_refused)
        {
            if (!_refused.Add($"{room}/{kind}")) return true;
        }

        Log.Write($"'{kind}' from room '{room}' refused; it belongs to the host");
        return true;
    }

    private void OnFaceMessage(FaceMessage inbound)
    {
        var message = inbound.Body;
        if (!message.TryGetProperty("type", out var typeNode)) return;

        var kind = typeNode.GetString();
        if (kind is null) return;

        Seen(inbound.From);
        var room = RoomOf(inbound.From);

        /* **`listen` means two different things, and which one depends on where it came
           from.** From the desk it is *"open the microphone on the machine she runs on"*, and
           that is hers to protect. From anywhere else it is *"transcribe what I am already
           sending you"* — a claim about the sender's own device, not about hers, so there is
           nothing there to refuse. Stage 15 item 3 concludes it should mean only the second
           thing everywhere, once the server holds no device at all.

           Checked before the authority table rather than inside it, because the table answers
           "may this room drive that hardware" and this is no longer a question about
           hardware. */
        /* **Stage 15 item 3, as far as one condition can carry it.** The rule above said
           `listen` should mean only the second thing *"once the server holds no device at
           all"*. It now means the second thing whenever the asking face **has a microphone of
           its own**, wherever it is standing — including the desk.

           So the desktop stops being a room and starts being a face that happens to be on the
           same box, which is the whole of the owner's rule. The server's own microphone is
           what a face with no device falls back to, rather than the first thing reached for;
           the hook is still there and is no longer the default path to it.

           `senses` is the signal because it is the honest one: a face that says it has a
           microphone is a face that will stream when asked, and one that says nothing gets
           the old behaviour unchanged. */
        var streamsItsOwn = _senses.TryGetValue(inbound.From, out var declared) &&
                            declared.Contains("mic");

        if (kind == "listen" && (room.Id != RoomId.Host || streamsItsOwn))
        {
            ToggleRoomListening(inbound.From, room);
            return;
        }

        if (HostOnly.Contains(kind) && room.Id != RoomId.Host)
        {
            Refuse(inbound.From, room.Id, kind);
            return;
        }

        switch (kind)
        {
            case "ready":
                _faceSpoke = true;
                _faceBuilt = message.TryGetProperty("faceBuilt", out var b) && b.ValueKind == JsonValueKind.True;

                /* A face declares its room and what it can do, and this is the only place
                   either is stated. **Not derived from the credential**, tempting as that is:
                   token means loopback means host would put two handsets in one room and make
                   a laptop on the LAN indistinguishable from a phone. */
                Join(inbound.From, RoomId.Named(Text(message, "room")));
                room = RoomOf(inbound.From);
                _senses[inbound.From] = Senses(message);

                /* **A room cannot be listening through a face that is not this one.**
                   Killing an app does not close its socket politely, so the server can hold a
                   dead face for as long as it takes a keepalive to give up — and during that
                   window the room is still "open" through it. The replacement face then
                   arrives, is told `listening: true` because the *room* is open, shows a lit
                   microphone button, and is ignored: its audio is not the open face's.
                   Blue button, deaf room, nothing in the log.

                   A fresh `ready` in a listening room means its client has changed, so the
                   honest state is off. One tap starts it again, against a face that exists. */
                if (_openFace is { } holder && _openRoom == room.Id && holder != inbound.From)
                    CloseRoomListening("the face that opened it was replaced");

                if (_faceBuilt) Log.Write("face ready; scene built");
                else Log.Warn("face ready but the scene did not build - check WebGL support and the browser console");
                Announce();

                // The host machine's ears, so only the host machine's face starts them.
                if (_config.ListenOnStart && room.Id == RoomId.Host) StartListening();
                break;

            case "faceError":
                Log.Error($"face error: {Text(message, "text")}");
                break;

            case "say":
                var text = Text(message, "text");
                if (!string.IsNullOrWhiteSpace(text)) RespondTo(room, text.Trim()).Forget("turn");
                break;

            case "listen":
                ToggleListening();
                break;

            case "hush":
                // Only the room she is talking to may stop her. Hushing from the next room
                // is reaching across and cutting somebody else off mid-sentence.
                if (room.Id == _attending) Hush();
                break;

            case "forget":
                room.History.Clear();
                ToRoom(room.Id, new { type = "cleared" });
                Notice(room, "Conversation forgotten.");
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

            /* Per room, not per being. "May she open a camera at all" is a question about a
               place: the gym phone and the desk should be able to answer it differently, and
               a phone answering yes must not open the lens on this machine.

               The host room's answer is still written to the config, because the config file
               belongs to this machine. Every other room's lives as long as she does. */
            case "setCamera":
                var seeing = !message.TryGetProperty("value", out var cam) || cam.ValueKind != JsonValueKind.False;
                room.Camera = seeing;

                if (room.Id == RoomId.Host) { _config.Camera = seeing; _config.Save(); }

                // Warn on the way up, wherever it came from: a camera coming on in someone's
                // home should leave a mark that is easy to find later, and the room is half
                // of what makes it findable.
                if (seeing) Log.Warn($"camera enabled in room '{room.Id}'");
                else Log.Write($"camera disabled in room '{room.Id}'");

                Announce();
                break;

            case "setCameraDevice":
                room.CameraDevice = Text(message, "value") ?? "";
                if (room.Id == RoomId.Host) { _config.CameraDevice = room.CameraDevice; _config.Save(); }
                Log.Write($"camera device in room '{room.Id}': {Named(room.CameraDevice)}");
                Announce();
                break;

            case "setWhisperCompute":
                SelectWhisperCompute(Text(message, "value") ?? "auto");
                break;

            case "setRoomHour":
                SelectRoomHour(message.TryGetProperty("value", out var h) && h.TryGetInt32(out var hour) ? hour : -1);
                break;

            case "selfTest":
                RunSelfTest(room).Forget("self-test");
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

    /// What a face claims it can do. An absent list is **not** an empty one: it means a face
    /// that predates the field, and treating that as "no camera" would stop `look` reaching
    /// the built-in page. See `Eyes` for how the difference is used.
    private static string[] Senses(JsonElement message)
    {
        if (!message.TryGetProperty("senses", out var node) || node.ValueKind != JsonValueKind.Array)
            return [];

        return node.EnumerateArray()
                   .Where(item => item.ValueKind == JsonValueKind.String)
                   .Select(item => item.GetString()!.Trim().ToLowerInvariant())
                   .Where(name => name.Length > 0)
                   .ToArray();
    }

    /// Bumped only for a breaking change to the message contract; see PROTOCOL.md.
    private const int ProtocolVersion = 1;

    /// **Per face, not broadcast.** It used to build one anonymous object and fan it out,
    /// which had already drawn blood once: the avatar URL had to be patched in the renderer
    /// because a single `hello` could not say different things to the built-in page and a
    /// phone. Now it must differ anyway — the room, what that face may drive, and what she
    /// is doing *in that room* are all different answers to different faces.
    private void Announce()
    {
        foreach (var face in KnownFaces()) Announce(face);
    }

    private void Announce(FaceId face) => _face.Send(Hello(face, RoomOf(face)), face);

    private object Hello(FaceId face, Room room) => new
    {
        type = "hello",
        protocol = ProtocolVersion,

        // Which room this face was put in, echoed back so a typo in `?room=` is visible
        // rather than mysterious, and `controls`: what it may drive. A page hides its
        // host-only switches when this says `room`. A hint, not the enforcement — see the
        // authority table in `OnFaceMessage`, which is where the refusal happens.
        room = room.Id.ToString(),
        controls = room.Id == RoomId.Host ? "host" : "room",

        // What she is doing and wearing *now*, in this face's room. `state` and `emotion`
        // are otherwise sent only when they change, so a face attaching to a session
        // already in progress had no way to know either — and an expression can sit
        // unchanged for a long time, so a renderer that assumed neutral would simply be
        // wrong until she next felt something. Found by the renderer contract check.
        state = room.State.ToString().ToLowerInvariant(),
        emotion = room.Mood.Name,
        emotionWeight = room.Mood.Weight,
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
        /* Per face, because "is she listening" is a question about a *room*. The desk asks
           about this machine's microphone; a handset asks whether the stream it is sending
           is being transcribed. One flag answered both and was wrong for one of them. */
        listening = room.Id == RoomId.Host ? _wantsToListen : _openRoom == room.Id,
        avatar = AvatarUrl(),
        avatarFile = _config.AvatarFile ?? "",
        avatars = Avatars(),
        roomHour = _config.RoomHour,
        music = _config.Music,
        musicAvailable = _music.IsRunning,
        camera = room.Camera,
        stats = _config.ShowStats,

        // What a face has to know to play her voice, announced rather than assumed. The
        // rate comes from the live voice's own model config, so it changes with the voice
        // — and `hello` is re-sent on every settings change, which carries the new rate
        // with it. A face must re-read this each time rather than caching it once.
        /* Whether the host will take audio *from* a face at all, so a client does not offer
           a microphone button that could only fail — the same courtesy `camera` already does.

           **"Will accept", not "has already started".** It used to report whether the
           recogniser was open, which made it false on every fresh session and told a handset
           its microphone button was useless — when in fact holding that button is what opens
           her ears. A phone read it correctly and hid a control that would have worked. */
        micAccepted = _ears is WhisperRecognizer || (_ears is null && WhisperWanted),

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
        cameraDevice = room.CameraDevice,
        whisperCompute = _config.WhisperCompute ?? "auto",

        // What she has been given hands for. Reported even though she cannot call them
        // yet, because "is the integration actually connected" is the first question
        // anyone will ask and it should not need a log to answer.
        toolServers = _tools.Providers.Select(p => new { name = p.Name, ready = p.IsReady }),

        dev = _config.DevPanel ?? string.Equals(_config.Profile, "dev", StringComparison.OrdinalIgnoreCase)
    };

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

    /// Answered to the room that asked. A report is a question about the machine, which any
    /// room may ask; it is not an answer the other rooms were waiting for.
    private async Task RunSelfTest(Room room)
    {
        ToRoom(room.Id, new { type = "diagnostics", running = true });

        try
        {
            var snapshot = Snapshot();
            var checks = await SelfTest.RunAsync(_config, snapshot);

            foreach (var check in checks.Where(c => !c.Ok))
                Log.Warn($"self-test: {check.Name} — {check.Detail}");

            ToRoom(room.Id, new
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
            ToRoom(room.Id, new { type = "diagnostics", running = false });
            Notice(room, "The self-test itself failed. The log has the detail.");
        }
    }

    /// The button that matters: a single zip that can be sent back from a machine nobody
    /// here can reach.
    ///
    /// **It used to open a Save dialog, and could not survive the split.** A dialog needs a
    /// dispatcher and somebody looking at it, and a server has neither — so the one control
    /// that exists for when she is broken would have been broken by moving her. It writes
    /// into her data folder instead and says where.
    ///
    /// That is better than what it replaces rather than a concession. The old dialog put the
    /// bundle only where the person at *this* machine chose, which is no use at all when the
    /// machine that needs diagnosing is in another room. The path goes back over the socket,
    /// so whoever asked is told regardless of where they are standing.
    internal async Task SaveDiagnosticsAsync()
    {
        string path;

        try
        {
            var folder = Directory.CreateDirectory(Path.Combine(Paths.DataDir, "diagnostics")).FullName;
            path = Path.Combine(folder, DiagnosticsBundle.SuggestedFileName());
        }
        catch (Exception ex)
        {
            Log.Error("diagnostics folder could not be made", ex);
            Notice(Host, $"Could not make a place to write diagnostics: {ex.Message}");
            return;
        }

        try
        {
            await Task.Run(() => DiagnosticsBundle.WriteAsync(path, _config, Snapshot()));
            Log.Write($"diagnostics written to {path}");
            ToRoom(RoomId.Host, new { type = "diagnosticsSaved", path });
            Notice(Host, $"Diagnostics saved to {path}. Open it and read README.txt before sending it on.");
        }
        catch (Exception ex)
        {
            Log.Error("diagnostics bundle failed", ex);
            Notice(Host, $"Could not write the diagnostics file: {ex.Message}");
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

    /// Whether her ears *could* take a face's microphone, as opposed to whether they have
    /// already been opened. `SystemSpeechRecognizer` is the machine's own and cannot be
    /// handed a stream; Whisper can.
    private bool WhisperWanted =>
        string.Equals(_config.Recognizer, "whisper", StringComparison.OrdinalIgnoreCase);

    /* Opening her ears, which is **not** the same act as listening with them.

       `listen` used to do both jobs, and item 9 making it host-only turned that into a real
       fault: a room face could never start the recogniser, so the microphone button restored
       in v0.25.0 could not work until somebody pressed listen at the desk. Reported from the
       handset as `micAccepted: false, ears: not started`.

       The two jobs pull apart cleanly. **Opening the recogniser is being-wide** — it is the
       same Whisper for every room, and a face taking the floor is an explicit request to be
       transcribed. **Opening this machine's microphone is a host-room device** and stays
       exactly where item 9 put it. */
    private Task<ISpeechRecognizer?>? _opening;
    private readonly object _earsGate = new();
    private bool _roomMusicStarted;

    /// Opens the recogniser before anybody asks, so the first person to speak is not paying
    /// for it. A server should be ready, not merely reachable.
    ///
    /// Deliberately *not* called from the constructor: this is a hosting decision rather than
    /// a session invariant, so `Being` makes it and the in-process checks — which build a real
    /// session dozens of times a run — do not load a 1.6 GB model to assert a routing rule.
    ///
    /// Failure is already handled inside: `OpenAndWireAsync` logs, notices and returns null,
    /// so a machine with no speech model still starts and simply cannot hear.
    internal async Task<string?> WarmEarsAsync()
    {
        if (!_config.OpenEarsOnStart) return null;

        Log.Write("opening her ears before anyone asks; the first press should not pay for it");
        var ears = await EnsureEarsAsync();
        if (ears is not null) Log.Write($"ears ready: {ears.EngineName}");
        return ears?.EngineName;
    }

    private Task<ISpeechRecognizer?> EnsureEarsAsync()
    {
        if (_ears is not null) return Task.FromResult<ISpeechRecognizer?>(_ears);

        // Two callers can arrive at once now — the desk pressing listen and a phone holding
        // its button — and Whisper must be opened once, not twice.
        lock (_earsGate)
        {
            if (_ears is not null) return Task.FromResult<ISpeechRecognizer?>(_ears);
            return _opening ??= OpenAndWireAsync();
        }
    }

    private async Task<ISpeechRecognizer?> OpenAndWireAsync()
    {
        try
        {
            var ears = await OpenEarsAsync();
            ears.Recognized += OnHeard;

            // The engine is this machine's, so its trouble is the host room's. What it is
            // *hearing* is not: while a face holds the floor these words came off a phone,
            // and a half-finished sentence belongs beside the mouth that said it.
            ears.Trouble += text => Notice(Host, text);
            ears.Hypothesised += partial =>
                ToRoom(EarsRoom, new { type = "caption", who = "You", text = partial, tentative = true });

            _ears = ears;
            Log.Write($"ears open: {ears.EngineName}");
            return ears;
        }
        catch (Exception ex)
        {
            Log.Error("cannot open ears", ex);
            Notice(Host, Explain(ex));
            return null;
        }
        finally
        {
            lock (_earsGate) _opening = null;
            Announce();
        }
    }

    /// The host room's own microphone, handed to the ears. Skipped while a face holds the
    /// floor: the desk starting to listen must not take the microphone out from under a
    /// phone mid-sentence.
    private void UseLocalMicrophone(ISpeechRecognizer ears)
    {
        if (ears is not WhisperRecognizer recogniser) return;

        _localMic ??= new LocalMicSource(_config.MicrophoneDevice);
        if (_floor is null) recogniser.UseSource(_localMic);
    }

    private void StartListening()
    {
        _wantsToListen = true;

        // Whisper's first listen may download the model; never block the message loop.
        _ = Task.Run(async () =>
        {
            var ears = await EnsureEarsAsync();

            if (ears is null)
            {
                _wantsToListen = false;
                if (Host.State is AgentState.Listening) SetState(Host, AgentState.Idle);
                return;
            }

            UseLocalMicrophone(ears);
            StartRoomMusic(ears);

            if (_wantsToListen) EngageEars();
        });
    }

    /// Room music. The microphone is already open and these are the very same frames the
    /// voice detector is reading, so hearing a speaker across the room costs one
    /// subscription and no extra capture. See ROADMAP.md 11a.
    private void StartRoomMusic(ISpeechRecognizer ears)
    {
        if (_roomMusicStarted || !_config.MusicFromRoom) return;
        if (ears is not WhisperRecognizer) return;

        _roomMusicStarted = true;

        /* The loopback wins when it has something.
           Both sources write to the same face state, so without a rule they fight: the
           tempo flickers between whatever this machine is playing and whatever is in the
           room, and neither reading is trustworthy. Loopback is the better witness — clean
           dynamics, no room, no gain control — so the microphone only speaks when it is
           silent. */
        _roomMusic.Changed += state =>
        {
            if (_music.State.Playing) return;
            ToRoom(RoomId.Host, new
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
            ToRoom(RoomId.Host, new { type = "music", beat = true });
        };

        /* Bound to the **local microphone**, not to whatever the recogniser is currently
           consuming.

           This used to be `whisper.Audio += _roomMusic.Push`, which was exactly right while
           there was only one microphone in the world. Once a face can hand the ears a phone,
           that subscription would quietly follow it — and she would report the tablet's
           kitchen radio as the music around *her*. Everything would appear to work.

           So the local source keeps running and is framed separately for music even while a
           face holds the floor for speech. Speech moves rooms; her sense of what is playing
           here does not. */
        _roomMusic.StartFromFrames(SileroVad.SampleRate);
        _localMic ??= new LocalMicSource(_config.MicrophoneDevice);
        _roomFramer.Frame += _roomMusic.Push;
        _localMic.Data += _roomFramer.Push;
        _localMic.Start();

        Log.Write("music: also listening for a beat in this room, through the local microphone");
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
            Notice(Host, "Whisper could not start; using the Windows recognizer for now.");
            return new SystemSpeechRecognizer(_config.RecognitionCulture);
        }
    }

    private void EngageEars()
    {
        _ears?.Start();
        _meter.Start();
        if (Host.State is AgentState.Idle) SetState(Host, AgentState.Listening);
        Announce();
    }

    private void StopListening()
    {
        _wantsToListen = false;
        _ears?.Stop();
        _meter.Stop();
        if (Host.State is AgentState.Listening) SetState(Host, AgentState.Idle);
        Announce();
    }

    private void OnHeard(string text, float confidence)
    {
        /* She is mid-turn, so this is somebody talking over her — and one of her cannot hold
           two threads at once, which is the same serialisation rooms already live under.
           **Said out loud now.** It was a bare `return`, and a room left listening makes it
           the most likely way an utterance dies: she answers at length, somebody speaks over
           the end of it, and nothing anywhere records that she heard and discarded them. */
        if (_responding)
        {
            Log.Write($"heard while she was still answering, so let go: {text}");
            return;
        }

        if (text.Length < _config.MinUtteranceChars) return;

        /* Peeked, not taken — `Consider` is what consumes it. A press is a *request*, so the
           thresholds that exist to keep the room's chatter out do not apply to it: somebody
           held a button and spoke, and dropping that on a confidence score is the same
           silence as ignoring them. Whisper is also least confident exactly where a phone is
           worst — a room mic at arm's length — so this fired most often on the face that
           could least afford it. */
        var asked = _owed?.Deliberate == true;

        if (confidence < _config.MinConfidence)
        {
            if (!asked)
            {
                Log.Debug($"discarded (confidence {confidence:0.00}): {text}");
                return;
            }

            // Info rather than debug: this one was asked for, so if she gets it wrong the
            // reason has to be findable without turning debug logging on.
            Log.Write($"low confidence ({confidence:0.00}) but it was asked for: {text}");
        }

        Consider(text).Forget("weighing something overheard");
    }

    /// Everything the ears produce comes through here; everything *typed* does not.
    /// If you took the trouble to type it, you meant it, and a gate that second-guesses
    /// a deliberate act is only ever wrong.
    private async Task Consider(string text)
    {
        /* The gate belongs to the room the words came from, and has no shared state with any
           other — the desk being mid-exchange must not make a second room's gate believe it
           is too, because "follow-up" is the rule that would then fire on a stranger.

           **Taken from `_owed` when a press is waiting to be attributed**, and only then from
           the live floor. Transcription finishes long after the button comes up, so reading
           `EarsRoom` here answered "host" for every sentence ever spoken into a phone. */
        var owed = _owed;
        _owed = null;

        var room = RoomNamed(owed?.Room ?? EarsRoom);

        /* A held button has already answered the question the gate asks, so it is not asked.
           This is what the comment here used to claim was happening. It was not: the gate
           judged push-to-talk too, and dropped anything under twelve characters — so
           pressing and saying "yes" reached her ears, her recogniser, and nothing else. */
        var verdict = owed?.Deliberate == true
            ? new Verdict(true, "asked for", TimeSpan.Zero)
            : await room.Gate.JudgeAsync(text);

        if (verdict.Answer)
        {
            await RespondTo(room, text);
            return;
        }

        // Never silently. "She ignored me" has to be a question with an answer, and the
        // face shows it faintly so the reason is visible without opening the log.
        Log.Write($"overheard ({verdict.Why}, {verdict.Cost.TotalMilliseconds:0} ms): {text}");
        ToRoom(room.Id, new { type = "overheard", text, why = verdict.Why });
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

    /// Who is holding the button, as opposed to who has the floor. They differ only while
    /// her ears are being opened — which can take a moment the first time — and the gap is
    /// why this exists: a person who lets go during it must not have the floor handed to
    /// them a second later.
    private FaceId? _pressing;

    private void OnTalking(FaceId from, bool holding)
    {
        if (holding)
        {
            _pressing = from;
            TakeFloorAsync(from).Forget("taking the floor");
            return;
        }

        if (_pressing == from) _pressing = null;
        if (_floor == from) ReleaseFloor("released");
    }

    private async Task TakeFloorAsync(FaceId from)
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

        /* Her attention, not just the floor. This is the same mechanism generalised: a
           button held in one room while she is mid-turn in another is refused for exactly
           the reason a second press is — she attends one room at a time, and pretending
           otherwise would need two Whispers, two voices and two brains in flight. */
        var asking = RoomOf(from);
        if (_responding && asking.Id != _attending)
        {
            _face.Send(new { type = "notice", text = "She is talking to someone else." }, from);
            Log.Write($"face {from} pressed from room '{asking.Id}' while she is in '{_attending}'; refused");
            return;
        }

        // Talking over her is an interrupt, not something to record on top of. `hush`
        // already does exactly the right thing — and only her attention is interruptible,
        // which by now is guaranteed to be this room's, because the refusal above ran first.
        if (RoomNamed(_attending).State is AgentState.Speaking or AgentState.Thinking) Hush();

        /* **Holding the button opens her ears if nobody has.**

           Until v0.25.1 this refused outright, and item 9 had quietly made that unreachable
           for the only faces that need it: the recogniser is started by `listen`, `listen` is
           host-only and rightly so, and therefore the microphone button restored on the
           handset in v0.25.0 could not work until somebody walked to the desk and pressed a
           different button. Reported from the phone as `micAccepted: false, ears: not
           started`.

           A held button is an explicit request to be transcribed, which is exactly the thing
           `listen` was also being asked to mean. Opening the recogniser is being-wide — one
           Whisper for every room. Opening *this machine's microphone* is not, and stays
           where item 9 put it. */
        if (_ears is not WhisperRecognizer)
        {
            if (!WhisperWanted)
            {
                _face.Send(new { type = "notice", text = "Her ears cannot take a microphone from another room." }, from);
                Log.Write($"face {from} pressed, but the recogniser is not Whisper; refused");
                return;
            }

            Log.Write($"face {from} pressed and her ears were shut; opening them");
            await EnsureEarsAsync();
        }

        if (_ears is not WhisperRecognizer recogniser)
        {
            _face.Send(new { type = "notice", text = "Her ears would not start." }, from);
            return;
        }

        // The first press may have spent a few seconds loading a model, and a person does not
        // hold a button while nothing happens. If they let go, nothing is taken.
        if (_pressing != from || _floor is not null) return;

        _floor = from;

        /* **A room that is already listening is already streaming through a source**, and
           building a second one would leave the first attached to nothing while its frames
           kept arriving — a press inside a listening room would go deaf. The floor is a claim
           on the stream, not a replacement for it. */
        var already = _openFace == from && _faceMic is not null;

        if (!already)
        {
            _faceMic = new FaceAudioSource($"a face ({from})");

            // The local microphone is muted for *speech* — she must not transcribe a blend of
            // this room and the other one. It keeps running for the room-music analyser, which
            // is a different question entirely. See the note on _localMic.
            recogniser.UseSource(_faceMic);
        }

        /* Then started, in that order and never the other way round: `Start` opens the local
           microphone if it has not been given a source, which is the one thing a phone
           pressing a button must never do.

           `Unmute` because a held button is an explicit request to be heard, and the mute
           left over from her last turn is only ever lifted when the *desk* is listening —
           so without this the ears would be open, the source right, and nothing heard. */
        recogniser.Unmute();
        recogniser.Start();
        if (!already) _faceMic!.Start();

        _floorTimeout = new System.Threading.Timer(
            _ => ReleaseFloor("timed out"), null, FloorLimit, Timeout.InfiniteTimeSpan);

        // Holding the button is where a turn begins, so her attention moves now rather than
        // when the sentence finally arrives — everything between the two, the partial
        // transcript most of all, has to be drawn in the room that is speaking.
        _attending = asking.Id;

        Log.Write($"face {from} has the floor, in room '{asking.Id}'");
    }

    // ---- always-on listening, in a room ----------------------

    /// <summary>Stage 14 item 6: a room asks to be listened to, or asks to stop.</summary>
    ///
    /// <remarks>
    /// **Not the floor**, for the reasons in the note on `_openFace`. A press still works on
    /// top of this and hands control back afterwards.
    ///
    /// **One at a time.** She has one recogniser, one source and one voice, so two rooms
    /// streaming at once is not a thing she can do — the same serialisation that already
    /// governs turns. The second asker is told out loud rather than left wondering, which is
    /// the rule item 9 set for every other collision.
    /// </remarks>
    private void ToggleRoomListening(FaceId from, Room room)
    {
        if (_openFace == from)
        {
            CloseRoomListening("asked to stop");
            return;
        }

        if (_openFace is { } other)
        {
            Notice(room, $"She is already listening in '{RoomOf(other).Id}'.");
            return;
        }

        /* The desk holds the recogniser's source while it is listening, and taking it would
           silently deafen somebody standing at the machine. Refused rather than stolen. */
        if (_wantsToListen)
        {
            Notice(room, "She is listening at the desk. Stop that first.");
            return;
        }

        OpenRoomListening(from, room).Forget("opening a room's ears");
    }

    private async Task OpenRoomListening(FaceId from, Room room)
    {
        var ears = await EnsureEarsAsync();

        if (ears is not WhisperRecognizer recogniser)
        {
            Notice(room, "Her ears would not start.");
            return;
        }

        // Asking twice while the model loaded, or walking away in the middle of it.
        if (_openFace is not null || !KnownFaces().Contains(from)) return;

        _openFace = from;
        _openRoom = room.Id;
        _faceMic = new FaceAudioSource($"a room ('{room.Id}')");

        recogniser.UseSource(_faceMic);
        recogniser.Unmute();
        recogniser.Start();
        _faceMic.Start();

        // The same thing the desk shows when its own ears open. Without it the room's pill
        // read "idle" while she was listening to it, which is the interface disagreeing with
        // the microphone — the one disagreement none of this is allowed to have.
        if (room.State is AgentState.Idle) SetState(room, AgentState.Listening);

        Log.Write($"listening continuously to room '{room.Id}' (face {from})");
        Announce();
    }

    /// <summary>Stop, and put the ears back where they were.</summary>
    ///
    /// **`UseSource` starts what it is given**, so handing back the local microphone when
    /// nobody at the desk asked for it would open this machine's microphone because somebody
    /// switched something off on a phone. That is the fault item 9 exists to prevent, and it
    /// has arrived through this door once already — see `ReleaseFloor`.
    private void CloseRoomListening(string why)
    {
        if (_openFace is not { } was) return;

        var room = _openRoom;
        _openFace = null;
        _openRoom = RoomId.Host;

        if (_ears is WhisperRecognizer recogniser)
        {
            // Whatever was mid-sentence belongs to the room that was speaking, not to
            // whoever speaks next. Same latch the floor uses, and for the same reason.
            if (recogniser.Flush()) _owed = (room, Deliberate: false);

            if (_wantsToListen)
            {
                if (_localMic is not null) recogniser.UseSource(_localMic);
            }
            else recogniser.Stop();
        }

        // Only if the floor is not using it. A press taken while the room was open owns the
        // source until it is released, and tearing it down here would cut that sentence off.
        if (_floor is null)
        {
            _faceMic?.Dispose();
            _faceMic = null;
        }

        var stopped = RoomNamed(room);
        if (stopped.State is AgentState.Listening) SetState(stopped, AgentState.Idle);

        Log.Write($"stopped listening to room '{room}' ({why}; face {was})");
        Announce();
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
            // Latched before the source is swapped away, because the words are still coming
            // and this is the last moment anything knows whose they are. See `_owed`.
            if (recogniser.Flush()) _owed = (RoomOf(held).Id, Deliberate: true);

            /* A room that was already listening keeps listening. The press was a louder claim
               laid on top of a quieter one, so letting go returns to it rather than ending
               both — otherwise pressing the button inside a room that is listening would
               *switch off* the thing the person had deliberately left on. */
            if (_openFace is not null)
            {
                // The stream never stopped; only the floor's claim on it did.
            }
            else if (_wantsToListen)
            {
                if (_localMic is not null) recogniser.UseSource(_localMic);
            }
            else
            {
                /* Nobody at the desk is listening, so the ears go quiet rather than being
                   handed this machine's microphone. **`UseSource` starts what it is given**,
                   so the obvious line here would open the host's microphone because a phone
                   let go of a button — which is precisely the fault item 9 exists to
                   prevent, arriving through a door item 9 did not fit a lock to. */
                recogniser.Stop();
            }
        }

        // Kept when a room is still streaming through it; it was never the floor's to own.
        if (_openFace is null)
        {
            _faceMic?.Dispose();
            _faceMic = null;
        }

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

    /// The face in a room that should be asked to look.
    ///
    /// This replaces `_lastSpokenThrough`, which was a stopgap that said so in its own
    /// comment. "Whoever last spoke" is not the question — the question is *who in this room
    /// has a camera*, and a face now answers it in `ready`.
    ///
    /// It matters concretely on Android: the native client owns the camera and the WebView
    /// panel cannot open one at all, because `getUserMedia` needs a secure context and the
    /// panel is served over plain HTTP. Without `senses` the host had a coin-flip chance of
    /// asking the half of the phone that physically cannot answer.
    ///
    /// **A face that declared no senses is a candidate of last resort, not a refusal.** That
    /// is what keeps `attach-face.ps1`, the checks and any renderer built before this field
    /// existed working exactly as they did.
    internal FaceId? EyesIn(RoomId room)
    {
        var faces = FacesIn(room);

        var claimed = faces.Where(f =>
            _senses.TryGetValue(f, out var senses) && senses.Contains("camera")).ToArray();

        // The built-in page first among equals, so the answer is stable rather than
        // whichever face happens to sort first in a dictionary.
        var page = _face.BuiltInFace;
        if (claimed.Length > 0)
            return page is { } known && claimed.Contains(known) ? known : claimed[0];

        var silent = faces.Where(f => !_senses.TryGetValue(f, out var s) || s.Length == 0).ToArray();
        if (silent.Length == 0) return null;

        return page is { } builtIn && silent.Contains(builtIn) ? builtIn : silent[0];
    }

    /// A still, but only if the question genuinely needs one and she is allowed.
    ///
    /// Three gates before a camera opens, and all three are cheap and legible: the
    /// setting is on **in this room**, the words need eyes, and the brain has any. Nothing
    /// here consults a model — the decision to open a camera in someone's home has to be
    /// auditable by reading it. See `Sight`.
    private async Task<string?> MaybeLookAsync(Room room, string userText, CancellationToken cancel)
    {
        if (!room.Camera || !Sight.WantsEyes(userText)) return null;

        // A brain with no eyes would only be told there was a picture it cannot see.
        if (_brain is not ClaudeBrain)
        {
            Notice(room, "She would need her Claude brain to see anything.");
            return null;
        }

        if (_looking is not null) return null;

        // One face, in the room that asked. Never the room next door: a question at the
        // desk must not open a lens on a handset in somebody's pocket.
        var target = EyesIn(room.Id);
        if (target is null)
        {
            Log.Warn($"look: no face in room '{room.Id}' has a camera, so she answers without eyes");
            return null;
        }

        var waiting = new TaskCompletionSource<string?>(TaskCreationOptions.RunContinuationsAsynchronously);
        _looking = (target.Value, waiting);

        try
        {
            Log.Write($"look: asking face {target.Value} in room '{room.Id}'");
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

    private async Task RespondTo(Room room, string userText)
    {
        /* **The single flag stays single, and that is the decision.** Making this re-entrant
           "because there are rooms now" is the concurrency change this deliberately defers:
           two rooms conversing at once means two Whispers and two synthesis pipelines on an
           eight-core box, and — more to the point — one being cannot hold two conversations
           at once. Pretending she can is a worse simulation, not a better one.

           So the other room is told, out loud, rather than left wondering. The same room
           pressing twice is left in silence, because it can see she is already answering. */
        if (_responding)
        {
            if (room.Id != _attending)
                Notice(room, "She is talking to someone else.");
            return;
        }

        if (!_brain.IsReady && _brain.NeedsApiKey)
        {
            Notice(room, "No API key yet. Paste one below and try again.");
            ToRoom(room.Id, new { type = "needKey" });
            return;
        }

        _responding = true;
        _attending = room.Id;
        _turn?.Cancel();
        _turn = new CancellationTokenSource();
        var cancel = _turn.Token;

        _ears?.Mute();
        _meter.Stop();

        /* Where her voice comes out. Silencing this machine's speakers is the whole of
           acceptance criterion 3 and it is not cosmetic: before this, a question asked on a
           handset was answered aloud in an empty house — and if the desk had opted into
           audio, in two rooms at once. The visemes and the streamed PCM are untouched; only
           the sound card is. See `IVoice.Aloud`. */
        var aloud = room.Id == RoomId.Host;
        _voice.Aloud = aloud;

        Log.Write(aloud
            ? $"turn in room '{room.Id}'; her voice plays on this machine"
            : $"turn in room '{room.Id}'; her voice goes only to that room, and this machine stays silent"
              + (_voice.AudioFormat is null ? " — and this voice cannot be streamed, so nobody will hear it" : ""));

        if (!aloud && _voice.AudioFormat is null)
            Notice(room, "Her Windows voice cannot leave this machine; you will see her words, not hear them.");

        ToRoom(room.Id, new { type = "caption", who = "You", text = userText });
        ToRoom(room.Id, new { type = "turn", who = "you", text = userText });
        SetState(room, AgentState.Thinking);

        var reply = new StringBuilder();

        try
        {
            /* `DateTimeOffset.Now`, read here rather than anywhere earlier: the answer has to
               be true at the moment she is asked, and `RoomHour` — which pins the *lighting*
               to an hour — must never reach this. The room can be told it is evening while
               she correctly says it is two in the morning. */
            var now = new Situation(
                Persona.Now(DateTimeOffset.Now, _music.State.Playing, _music.State.Bpm),
                await MaybeLookAsync(room, userText, cancel));

            await foreach (var sentence in _brain.RespondAsync(room.History, userText, now, cancel))
            {
                if (cancel.IsCancellationRequested) break;

                reply.Append(reply.Length > 0 ? " " : "").Append(sentence);
                ToRoom(room.Id, new { type = "caption", who = "Octavia", text = reply.ToString() });
                Feel(room, Moods.Read(sentence));
                _voice.Say(sentence);
            }

            if (reply.Length > 0)
                ToRoom(room.Id, new { type = "turn", who = "octavia", text = reply.ToString() });
            else if (!cancel.IsCancellationRequested)
                Notice(room, "She had nothing to say.");
        }
        catch (OperationCanceledException)
        {
            // hushed mid-thought
        }
        catch (Exception ex)
        {
            Log.Error("turn failed", ex);
            Notice(room, Explain(ex));
            ToRoom(room.Id, new { type = "caption", who = "", text = "Something went wrong reaching the model." });
        }
        finally
        {
            _responding = false;
            room.Gate.Answered();
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

        var spokenIn = RoomNamed(_attending);

        // The turn is over; her face should not keep the last sentence's mood on it.
        Feel(spokenIn, Mood.Neutral);

        // Her voice belongs to this machine again the moment the turn ends, so nothing that
        // happens next — a hotkey, an utterance into the desk microphone — can be swallowed
        // by a flag left over from a phone conversation.
        _voice.Aloud = true;

        if (_wantsToListen)
        {
            _ears?.Unmute();
            _meter.Start();
        }

        /* The host room follows this machine's ears. Any other room simply goes quiet: its
           microphone is push-to-talk and belongs to the client, so "listening" would be a
           claim about a device this process does not own. */
        SetState(Host, _wantsToListen ? AgentState.Listening : AgentState.Idle);
        if (spokenIn.Id != RoomId.Host) SetState(spokenIn, AgentState.Idle);
    }

    /// Sent only when it changes: an expression is a change of face, and repeating one
    /// every sentence would keep restarting the movement towards it.
    ///
    /// Per room, because it drives the avatar. A global mood would put an expression on the
    /// phone's face that was caused by a conversation in a different building.
    private void Feel(Room room, Mood mood)
    {
        if (mood.Name == room.Mood.Name && Math.Abs(mood.Weight - room.Mood.Weight) < 0.01) return;
        room.Mood = mood;
        ToRoom(room.Id, new { type = "emotion", value = mood.Name, weight = mood.Weight });
    }

    private void SetState(Room room, AgentState state)
    {
        if (room.State == state) return;
        room.State = state;

        /* Everything she says comes back through the loopback a few milliseconds later.
           Holding the analysis while she talks is what stops her hearing herself and
           deciding it is music; the tempo she already had keeps running underneath.

           A host concern, so it follows the room she is *attending* rather than any room
           that happens to change state — the loopback only ever hears this machine. */
        if (room.Id == _attending) _music.Hold = state is AgentState.Speaking;

        ToRoom(room.Id, new { type = "state", value = state.ToString().ToLowerInvariant() });
    }

    /// Notices are the things she thought were worth interrupting for, so they belong in
    /// the log as well as on her face — by the time a bundle arrives, the one that
    /// mattered has long since faded off the screen.
    /// Everywhere. For the things that are about *her* rather than about a room — a voice
    /// engine that would not start, a key that was stored — because every face is wearing
    /// the result.
    private void Notice(string text)
    {
        Log.Write($"notice: {text}");
        _face.Send(new { type = "notice", text });
    }

    /// One room. For the things that answer something that room asked.
    private void Notice(Room room, string text)
    {
        Log.Write($"notice ({room.Id}): {text}");
        ToRoom(room.Id, new { type = "notice", text });
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
        foreach (var room in _rooms.Values) room.Dispose();
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
