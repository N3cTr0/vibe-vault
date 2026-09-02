---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\RoomChecks.cs
---

# tools\EarsTest\RoomChecks.cs

```csharp
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Text.Json;
using Octavia;
using Octavia.Core;
using Octavia.Face;

/// Stage 14 item 9 — two rooms, driven through a real `OctaviaSession`.
///
/// Two faults were being fixed and both are asserted here. One was security-shaped: **no
/// `set*` case looked at where a message came from**, so a phone at the gym could open the
/// microphone in an empty house. The other was architectural: one `Conversation` and
/// untargeted `caption`/`turn`/`state`/`emotion`, so every face was a window onto the same
/// room — typing at her on a handset put the words on the desk and played the answer where
/// nobody was standing.
///
/// The session is driven through a recording transport and a stub local model, so the ten
/// acceptance criteria the Android side will check from a handset are checked here first,
/// in-process, with no phone and no API key.
///
/// **The conversations happen between two non-host rooms on purpose.** Her voice is silenced
/// in any room but the host's — that is the fix itself — so running them in `phone` and
/// `kitchen` proves the routing without the machine talking out loud through the suite.
internal static class RoomChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;

        void Check(string name, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}{(!ok && detail.Length > 0 ? $" — {detail}" : "")}");
            if (!ok) failures++;
        }

        /* Her real config and her real log are both redirected. `setCamera` and
           `setWhisperCompute` are *supposed* to write themselves down, and a check that
           leaves "camera enabled" in her log — or in her settings — hands somebody an
           incident that never happened. The lesson is already written down in this project;
           it is applied here rather than relearned. */
        var beforeConfig = Environment.GetEnvironmentVariable("OCTAVIA_CONFIG");
        var beforeLog = Environment.GetEnvironmentVariable("OCTAVIA_LOG");
        var scratchConfig = Path.Combine(Path.GetTempPath(), $"octavia-rooms-{Guid.NewGuid():N}.json");
        var scratchLog = Path.Combine(Path.GetTempPath(), $"octavia-rooms-{Guid.NewGuid():N}.log");
        Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", scratchConfig);
        Environment.SetEnvironmentVariable("OCTAVIA_LOG", scratchLog);

        using var model = new StubModel();

        try
        {
            var config = new OctaviaConfig
            {
                Profile = "test",
                Brain = "local",
                LocalEndpoint = model.Endpoint,
                LocalModel = "stub",
                VoiceEngine = "windows",
                Music = false,
                MusicFromRoom = false,
                ListenOnStart = false,
                Camera = false,
                Gate = "off",
                WhisperCompute = "auto",

                // Whisper, and the smallest model, because one check really does open her
                // ears — a room face taking the floor is what starts the recogniser now.
                Recognizer = "whisper",
                WhisperModel = "tiny.en"
            };

            var face = new RecordingFace();
            using var session = new OctaviaSession(config, face);

            var desk = face.Page;
            var phone = FaceId.New();
            var panel = FaceId.New();
            var kitchen = FaceId.New();

            // --- who is where -------------------------------------------
            // The desk names no room and claims no senses, which is exactly what every
            // renderer written before this stage does. It must land in the host room.
            face.From(desk, new { type = "ready", faceBuilt = true });
            face.From(phone, new { type = "ready", faceBuilt = true, room = "phone", senses = new[] { "mic", "camera" } });
            face.From(panel, new { type = "ready", faceBuilt = true, room = "phone", senses = Array.Empty<string>() });
            face.From(kitchen, new { type = "ready", faceBuilt = true, room = "kitchen", senses = new[] { "mic" } });

            // --- 9. a face that names no room behaves exactly as it did --
            var deskHello = face.Last(desk, "hello");
            Check("a face that names no room is in the host room",
                Str(deskHello, "room") == "host", Str(deskHello, "room") ?? "no hello at all");
            Check("the host room's face is told it drives the host",
                Str(deskHello, "controls") == "host", Str(deskHello, "controls") ?? "—");

            var phoneHello = face.Last(phone, "hello");
            Check("a face that names a room is put in it",
                Str(phoneHello, "room") == "phone", Str(phoneHello, "room") ?? "—");
            Check("a face outside the host room drives only its room",
                Str(phoneHello, "controls") == "room", Str(phoneHello, "controls") ?? "—");

            // `hello` used to be one anonymous object broadcast to everyone, which could
            // not have answered these two differently at all.
            Check("hello is per face, not per session",
                Str(deskHello, "controls") != Str(phoneHello, "controls"));

            // --- 1. the phone cannot drive this machine ------------------
            // The whole of the first fault: `listen` toggles the host's microphone and
            // nothing looked at who sent it. Hiding the button was never the fix.
            face.Clear();
            face.From(phone, new { type = "setWhisperCompute", value = "cpu" });

            Check("a host-only message from another room does not act",
                config.WhisperCompute == "auto", $"it became '{config.WhisperCompute}'");
            Check("a host-only message from another room is refused out loud",
                face.Any(phone, "notice"), "ignored in silence, which reads as broken");
            Check("a refusal announces nothing", !face.Any(phone, "hello"));

            /* `listen` is deliberately **not** in this list any more, and its absence is the
               whole of Stage 14 item 6. From the desk it means "open the microphone on the
               machine she runs on" and is hers to protect; from a room it means "transcribe
               what I am already sending you", which is a claim about the sender's own device
               and has nothing here to refuse. It is checked on its own below. */
            foreach (var kind in new[]
                     { "setMicrophone", "setOutput", "setMusic", "openDataFolder", "saveDiagnostics" })
            {
                face.Clear();
                face.From(phone, new { type = kind, value = "" });
                Check($"'{kind}' from another room is refused", face.Any(phone, "notice"));
            }

            /* Item 6: a room asking to listen is answered rather than refused — and, more to
               the point, **it must not touch this machine's microphone on the way**. That is
               the fault item 9 exists to prevent arriving through the one door item 6 opens,
               so it is worth asserting both halves rather than only the friendly one. */
            face.Clear();
            face.From(phone, new { type = "listen" });

            Check("'listen' from another room is not refused any more",
                !face.Any(phone, "notice"),
                "item 6 gave a room its own ears; something still says they belong to the desk");

            Check("...and it did not open this machine's microphone",
                !Log.Tail(20).Any(line => line.Contains("listening on '")),
                "a phone asking to be heard opened the desk's microphone");

            // Left as it was found, so the checks after this one are not standing in a room
            // that is quietly still streaming.
            face.Clear();
            face.From(phone, new { type = "listen" });

            Check("the microphone was never changed from the phone",
                config.MicrophoneDevice == "", config.MicrophoneDevice);
            Check("music listening was never changed from the phone",
                !config.Music, config.Music.ToString());

            // ...and the same message from the host room is simply obeyed. A guard that
            // refuses everybody is not a guard, it is a broken feature.
            face.Clear();
            face.From(desk, new { type = "setWhisperCompute", value = "cpu" });
            Check("the same message from the host room acts",
                config.WhisperCompute == "cpu", $"it stayed '{config.WhisperCompute}'");

            // --- settings that change *her* reach every room -------------
            face.Clear();
            face.From(phone, new { type = "setStats", value = false });
            Check("a setting that changes her is allowed from any room",
                !config.ShowStats, "the phone was refused");
            Check("...and every room is told, because every face is wearing it",
                face.Last(desk, "hello") is { } shown && !shown.GetProperty("stats").GetBoolean(),
                "the desk was never told");

            // --- 10. the camera is answered per room ---------------------
            face.Clear();
            face.From(phone, new { type = "setCamera", value = true });

            Check("enabling the camera in one room enables it there",
                face.Last(phone, "hello") is { } onPhone && onPhone.GetProperty("camera").GetBoolean());
            Check("...and does not enable it on the desktop",
                face.Last(desk, "hello") is { } onDesk && !onDesk.GetProperty("camera").GetBoolean());
            Check("...and does not write itself into this machine's settings",
                !config.Camera, "a phone changed the host config");
            Check("...and leaves a mark that names the room",
                Log.Tail(20).Any(line => line.Contains("camera enabled in room 'phone'")),
                "nothing in the log says where the camera came on");

            // --- 7. look goes to the half of the phone that has a camera --
            /* Asserted against the rule rather than through a turn, because these checks run
               on a local brain and `MaybeLookAsync` requires the Claude one.

               This comment used to say "and there is no key on this machine", inherited from
               item 1's landed note. **That was never true** — the key was there, the default
               profile is simply local. The round trip was walked from a handset on
               `--profile cloud` the same day, and this stays as the cheap check that the
               *choice of face* is right, which is the part a phone cannot easily prove.

               It matters concretely on Android. The native client owns the camera and the
               WebView panel cannot open one at all, because `getUserMedia` needs a secure
               context and the panel is served over plain HTTP; without `senses` the host had
               a coin-flip chance of asking the half of the phone that physically cannot
               answer. */
            Check("look goes to the face in the room that claims a camera",
                session.EyesIn(new RoomId("phone")) == phone,
                $"it chose {session.EyesIn(new RoomId("phone"))}");
            Check("look never leaves the room that asked",
                session.EyesIn(new RoomId("kitchen")) is null,
                "a room with no camera was offered one from next door");
            Check("a face that predates `senses` is still asked",
                session.EyesIn(RoomId.Host) == desk, "the built-in page stopped being asked");

            // --- 2, 4, 8. one room's conversation is its own --------------
            model.Reply = "Hello from the phone.";
            await Say(face, phone, "are you there");

            Check("she answers the room that asked",
                face.Text(phone, "turn", "who") == "octavia", "no answer came back");
            Check("the phone's words go on the phone's screen", face.Any(phone, "caption"));
            Check("both of a room's faces see the same conversation",
                face.Any(panel, "caption") && face.Any(panel, "turn"),
                "the WebView panel was left out of its own room");

            Check("nothing of it reaches the desktop's placard", !face.Any(desk, "caption"));
            Check("nothing of it reaches the desktop's transcript", !face.Any(desk, "turn"));
            Check("the desktop's state pill never moves", !face.Any(desk, "state"),
                "the desk was told she was thinking about somebody else's question");
            Check("the desktop's expression never moves", !face.Any(desk, "emotion"));

            // --- 3. her voice does not play in an empty room --------------
            Check("the log says where her voice went",
                Log.Tail(60).Any(line => line.Contains("her voice goes only to that room")),
                "the audio route is not in the log at all");

            // --- 5. forgetting is per room --------------------------------
            await Say(face, phone, "and again");
            Check("a room keeps its own history",
                model.LastUserTurns == 2, $"the model was sent {model.LastUserTurns} user turns");

            await Say(face, kitchen, "different room, first thing said");
            Check("another room starts from nothing",
                model.LastUserTurns == 1,
                $"the kitchen inherited {model.LastUserTurns - 1} turns from the phone");

            face.Clear();
            face.From(phone, new { type = "forget" });
            Check("clearing a room clears that room's screen", face.Any(phone, "cleared"));
            Check("...and its other face too", face.Any(panel, "cleared"));
            Check("...and leaves the desktop's alone", !face.Any(desk, "cleared"));

            await Say(face, phone, "starting over");
            Check("forgetting really emptied that room's history",
                model.LastUserTurns == 1, $"{model.LastUserTurns} user turns survived");

            await Say(face, kitchen, "still here though");
            Check("...and left the other room's history untouched",
                model.LastUserTurns == 2,
                $"the kitchen was cleared too, at {model.LastUserTurns} turns");

            // --- 6. she attends one room at a time ------------------------
            /* The mechanism `TakeFloor` already used for the floor, generalised from *the
               floor* to *her attention*. Deliberately not made re-entrant: two rooms at once
               means two Whispers and two synthesis pipelines, and one being holding two
               conversations is a worse simulation rather than a better one. */
            face.Clear();
            model.Hold = TimeSpan.FromSeconds(2);
            model.Reply = "One at a time.";

            var busy = Say(face, phone, "take your time");
            Check("a turn starts in the room that asked", await Wait(() => face.Any(phone, "state")));

            face.From(kitchen, new { type = "say", text = "me too" });
            Check("the other room is told, rather than ignored",
                face.Text(kitchen, "notice", "text") == "She is talking to someone else.",
                face.Text(kitchen, "notice", "text") ?? "nothing came back");
            Check("...and its words never reach her", !face.Any(kitchen, "turn"));

            await busy;
            Check("the interrupted room's turn still completes",
                face.Text(phone, "turn", "who") == "octavia",
                "cutting in cost the first room its answer");

            model.Hold = TimeSpan.Zero;

            /* --- a room face can start her ears ---------------------------
               Item 9 made `listen` host-only, correctly. What nobody noticed is that
               `listen` was doing two jobs: opening this machine's microphone, which is a
               host-room device, and starting the **recogniser**, which is being-wide — one
               Whisper for every room. So the microphone button restored on the handset in
               v0.25.0 could not work until somebody walked to the desk and pressed a
               different button. Reported from the phone as `micAccepted: false`. */
            face.Clear();
            face.From(phone, new { type = "ready", faceBuilt = true, room = "phone", senses = new[] { "mic", "camera" } });

            Check("her ears are offered before anything has started them",
                face.Last(phone, "hello") is { } fresh && fresh.GetProperty("micAccepted").GetBoolean(),
                "a handset was told its microphone button could only fail");

            var modelFile = Octavia.Senses.WhisperModelStore.PathFor(config.WhisperModel);
            if (File.Exists(modelFile))
            {
                /* The real thing, because the model is already on this machine — pressing
                   the button on a session where nothing has ever listened. `tiny.en` is
                   chosen for the suite's sake; the path is identical for any of them. */
                face.From(phone, new { type = "talking", value = true });

                var heard = await Wait(() =>
                    Log.Tail(12).Any(line => line.Contains($"face {phone} has the floor")), 40000);

                Check("holding the button opens her ears and takes the floor", heard,
                    "the floor was never taken, so nothing the phone said could be heard");

                face.From(phone, new { type = "talking", value = false });
                await Wait(() => Log.Tail(6).Any(line => line.Contains("released the floor")));

                // And it must not have opened the host's microphone on the way: `UseSource`
                // starts what it is given, so the release path is where that would happen.
                Check("...without opening this machine's microphone",
                    !Log.Tail(30).Any(line => line.Contains("listening on '")),
                    "a phone letting go of a button opened the desk's microphone");
            }
            else
            {
                Console.WriteLine($"  ..   skipped the live floor check: no {config.WhisperModel} model on this machine");
            }

            // --- a face that leaves stops being addressable ---------------
            face.Leave(panel);
            await Say(face, phone, "just us now");
            Check("a face that left receives nothing", !face.Any(panel, "caption"));
            Check("...and the rest of its room is unaffected", face.Any(phone, "caption"));
        }
        finally
        {
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", beforeConfig);
            Environment.SetEnvironmentVariable("OCTAVIA_LOG", beforeLog);
            try { File.Delete(scratchConfig); } catch { /* a temp file that outlives the run is harmless */ }
            try { File.Delete(scratchLog); } catch { /* likewise */ }
        }

        return failures;
    }

    /// Says something and waits for the answer to land, so the assertions after it are
    /// reading a finished exchange rather than racing one.
    ///
    /// It waits for the *turn* and nothing else. Waiting on "a turn or any notice" looked
    /// equivalent and was not: a room she cannot be heard in is told so at the top of the
    /// turn — "her Windows voice cannot leave this machine" — which ended the wait before
    /// she had said a word, and five checks failed for a reason that was in the test.
    ///
    /// **And the transcript is not the end of the turn.** `turn` is sent inside the `try`,
    /// while `_responding` is cleared in the `finally` after her voice drains — so a check
    /// that spoke again the moment a transcript appeared was refused with "she is talking to
    /// someone else", correctly, by code that was working. The room going idle is the signal
    /// that she is free.
    private static async Task Say(RecordingFace face, FaceId from, string text)
    {
        face.Clear();
        face.From(from, new { type = "say", text });
        await Wait(() => face.Text(from, "turn", "who") == "octavia" &&
                         face.Text(from, "state", "value") == "idle");
    }

    private static string? Str(JsonElement? message, string property) =>
        message is { } node && node.TryGetProperty(property, out var value) &&
        value.ValueKind == JsonValueKind.String
            ? value.GetString()
            : null;

    private static async Task<bool> Wait(Func<bool> until, int millis = 10000)
    {
        var deadline = DateTime.UtcNow.AddMilliseconds(millis);
        while (DateTime.UtcNow < deadline)
        {
            if (until()) return true;
            await Task.Delay(20);
        }
        return until();
    }
}

/// A face that records instead of rendering, so what the session *addressed* can be read
/// back afterwards. Everything here is about who a message reached; nothing draws anything.
internal sealed class RecordingFace : IFaceTransport
{
    private readonly record struct Sent(FaceId? To, string Type, JsonElement Body);

    private readonly List<Sent> _sent = [];

    /// The built-in page, minted for the same reason `WebViewFaceTransport` mints one: this
    /// transport is exactly one renderer and it still needs an address.
    public FaceId Page { get; } = FaceId.New();

    public FaceId? BuiltInFace => Page;
    public FaceStatus Status => new(Page: true, SocketBound: false, Port: 0, SocketFaces: 0);

    public event Action<FaceMessage>? MessageReceived;
    public event Action<FaceId>? FaceDeparted;

    // Nothing pushes microphone audio in these checks; the event is part of the seam.
#pragma warning disable CS0067
    public event Action<FaceId, byte[]>? AudioReceived;
#pragma warning restore CS0067

    /// Who her voice was addressed to on the last frame. Empty is a meaningful answer: it is
    /// what a room with nobody in it looks like.
    public IReadOnlyCollection<FaceId> LastHeardBy { get; private set; } = [];

    public void Send(object message, FaceId? to = null) => Record(to, message);

    public void SendMany(object message, IReadOnlyCollection<FaceId> to)
    {
        foreach (var face in to) Record(face, message);
    }

    public void SendAudio(ReadOnlyMemory<byte> pcm, IReadOnlyCollection<FaceId> to) => LastHeardBy = to;

    /// Which faces have subscribed to her voice, for the checks that care.
    ///
    /// Empty by default, so every existing room check sees the behaviour it was written
    /// against: nobody is playing her, therefore the sound card still does.
    public HashSet<FaceId> Hearing { get; } = [];

    public bool AnyWantsAudio(IReadOnlyCollection<FaceId> faces) => faces.Any(Hearing.Contains);

    private void Record(FaceId? to, object message)
    {
        var json = JsonSerializer.Serialize(message, FaceHub.Json);
        using var doc = JsonDocument.Parse(json);
        var body = doc.RootElement.Clone();
        var type = body.TryGetProperty("type", out var node) ? node.GetString() ?? "" : "";

        lock (_sent) _sent.Add(new Sent(to, type, body));
    }

    /// Something a face said to the host. Synchronous, like the real transports: the session
    /// handles it on the thread that raised it.
    public void From(FaceId face, object message)
    {
        var json = JsonSerializer.Serialize(message, FaceHub.Json);
        using var doc = JsonDocument.Parse(json);
        MessageReceived?.Invoke(new FaceMessage(face, doc.RootElement.Clone()));
    }

    public void Leave(FaceId face) => FaceDeparted?.Invoke(face);

    public void Clear()
    {
        lock (_sent) _sent.Clear();
    }

    /// Everything that reached this face. A null recipient still means everyone, which is
    /// what the being-wide messages use.
    private List<Sent> Reaching(FaceId face)
    {
        lock (_sent) return _sent.Where(s => s.To is null || s.To == face).ToList();
    }

    public bool Any(FaceId face, string type) => Reaching(face).Any(s => s.Type == type);

    public JsonElement? Last(FaceId face, string type)
    {
        var found = Reaching(face).LastOrDefault(s => s.Type == type);
        return found.Type == type ? found.Body : null;
    }

    public string? Text(FaceId face, string type, string property) =>
        Last(face, type) is { } body && body.TryGetProperty(property, out var value) &&
        value.ValueKind == JsonValueKind.String
            ? value.GetString()
            : null;
}

/// A local-model server that says one sentence, in the shape `LocalBrain` reads.
///
/// Raw sockets rather than `HttpListener`, which is the choice `WebSocketFaceServer` made
/// and for a similar reason: no URL reservation, no elevation, and the whole exchange is
/// readable in forty lines.
internal sealed class StubModel : IDisposable
{
    private readonly TcpListener _listener;
    private readonly CancellationTokenSource _stopping = new();

    public StubModel()
    {
        _listener = new TcpListener(IPAddress.Loopback, 0);
        _listener.Start();
        Port = ((IPEndPoint)_listener.LocalEndpoint).Port;
        _ = AcceptAsync();
    }

    public int Port { get; }
    public string Endpoint => $"http://127.0.0.1:{Port}/v1";

    public string Reply { get; set; } = "All right.";

    /// How long to sit on a request before answering. It is what makes "she is talking to
    /// someone else" observable at all: without it a turn is over before the next room can
    /// open its mouth.
    public TimeSpan Hold { get; set; } = TimeSpan.Zero;

    /// How many `user` messages the last request carried — the length of that room's
    /// history, read from the one place it is externally visible.
    public int LastUserTurns { get; private set; }

    private async Task AcceptAsync()
    {
        while (!_stopping.IsCancellationRequested)
        {
            TcpClient client;
            try { client = await _listener.AcceptTcpClientAsync(_stopping.Token); }
            catch { return; }

            _ = ServeAsync(client);
        }
    }

    private async Task ServeAsync(TcpClient client)
    {
        using (client)
        {
            try
            {
                var stream = client.GetStream();
                var head = new StringBuilder();

                while (!head.ToString().EndsWith("\r\n\r\n", StringComparison.Ordinal))
                {
                    var next = await ReadByteAsync(stream);
                    if (next < 0) return;
                    head.Append((char)next);
                }

                /* **`Content-Length` is not always there, and assuming it was is what made
                   the first version of this read nothing at all.** `PostAsJsonAsync` sends a
                   `JsonContent`, which cannot compute its own length up front, so HttpClient
                   falls back to `Transfer-Encoding: chunked`. A stub that only understands
                   one framing silently reports an empty request — and the check that reads
                   the history out of it then fails for a reason that has nothing to do with
                   rooms. */
                var headers = head.ToString();
                var chunked = headers.Contains("Transfer-Encoding: chunked", StringComparison.OrdinalIgnoreCase);

                var length = 0;
                foreach (var line in headers.Split("\r\n"))
                    if (line.StartsWith("Content-Length:", StringComparison.OrdinalIgnoreCase))
                        length = int.Parse(line["Content-Length:".Length..].Trim());

                var body = chunked
                    ? await ReadChunkedAsync(stream)
                    : await ReadExactlyAsync(stream, length);

                if (body is null) return;

                Remember(Encoding.UTF8.GetString(body));

                if (Hold > TimeSpan.Zero) await Task.Delay(Hold, _stopping.Token);

                var chunk = JsonSerializer.Serialize(new
                {
                    choices = new[] { new { delta = new { content = Reply } } }
                });

                var payload = Encoding.UTF8.GetBytes($"data: {chunk}\n\ndata: [DONE]\n\n");
                var header = Encoding.UTF8.GetBytes(
                    "HTTP/1.1 200 OK\r\nContent-Type: text/event-stream\r\n" +
                    $"Content-Length: {payload.Length}\r\nConnection: close\r\n\r\n");

                await stream.WriteAsync(header, _stopping.Token);
                await stream.WriteAsync(payload, _stopping.Token);
                await stream.FlushAsync(_stopping.Token);
            }
            catch
            {
                // A test double that throws on the way out is noise, not a finding.
            }
        }
    }

    private async Task<int> ReadByteAsync(NetworkStream stream)
    {
        var one = new byte[1];
        return await stream.ReadAsync(one, _stopping.Token) == 0 ? -1 : one[0];
    }

    private async Task<byte[]?> ReadExactlyAsync(NetworkStream stream, int length)
    {
        var buffer = new byte[length];
        var read = 0;

        while (read < length)
        {
            var got = await stream.ReadAsync(buffer.AsMemory(read), _stopping.Token);
            if (got == 0) return null;
            read += got;
        }

        return buffer;
    }

    /// Size in hex, CRLF, that many bytes, CRLF, until a size of zero. Trailers are not read
    /// because HttpClient does not send any.
    private async Task<byte[]?> ReadChunkedAsync(NetworkStream stream)
    {
        var body = new List<byte>();

        while (true)
        {
            var line = new StringBuilder();
            while (!line.ToString().EndsWith("\r\n", StringComparison.Ordinal))
            {
                var next = await ReadByteAsync(stream);
                if (next < 0) return null;
                line.Append((char)next);
            }

            var size = Convert.ToInt32(line.ToString().Trim().Split(';')[0], 16);

            if (size == 0)
            {
                /* The blank line that closes the trailer section. **Leaving it unread is not
                   harmless**: bytes still sitting in the receive buffer at close time make
                   Windows send an RST instead of a FIN, the client throws away the response
                   it had already been given, and `PostAsJsonAsync` fails with "error while
                   copying content to a stream" — which reads exactly like a model server
                   that is not there. */
                await ReadByteAsync(stream);
                await ReadByteAsync(stream);
                return body.ToArray();
            }

            var chunk = await ReadExactlyAsync(stream, size);
            if (chunk is null) return null;
            body.AddRange(chunk);

            // The CRLF that closes the chunk.
            await ReadByteAsync(stream);
            await ReadByteAsync(stream);
        }
    }

    private void Remember(string request)
    {
        try
        {
            using var doc = JsonDocument.Parse(request);
            LastUserTurns = doc.RootElement.GetProperty("messages").EnumerateArray()
                .Count(m => m.TryGetProperty("role", out var role) && role.GetString() == "user");
        }
        catch (JsonException)
        {
            LastUserTurns = -1;
        }
    }

    public void Dispose()
    {
        _stopping.Cancel();
        _listener.Stop();
        _stopping.Dispose();
    }
}
```
