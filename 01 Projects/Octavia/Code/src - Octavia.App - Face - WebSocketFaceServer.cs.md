---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\WebSocketFaceServer.cs
---

# src\Octavia.App\Face\WebSocketFaceServer.cs

```csharp
using System.Collections.Concurrent;
using System.Net;
using System.Net.Sockets;
using System.Net.WebSockets;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Threading.Channels;
using Octavia.Core;

namespace Octavia.Face;

/// A loopback WebSocket server so any renderer can be a face — the built-in page, a
/// browser on a tablet, or an Unreal application later. Protocol in PROTOCOL.md.
///
/// Raw TcpListener rather than HttpListener: HttpListener needs a urlacl reservation
/// for anything but the default namespace, which would mean running elevated or a
/// setup step. The handshake is a dozen lines and WebSocket.CreateFromStream does the
/// framing, so nothing here is hand-rolled that matters.
internal sealed class WebSocketFaceServer : IDisposable
{
    private const string HandshakeGuid = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

    /// Set once the query string has proved who a browser is, so the page's own scripts,
    /// styles and characters can be fetched without a credential the browser has no way
    /// to attach. `HttpOnly` because nothing in `wwwroot` reads these — only the socket
    /// does — and script that cannot see a secret cannot leak one.
    private const string CookieName = "octavia_token";
    private const string KeyCookieName = "octavia_key";

    /// One thing to write, and what kind of frame it is. A binary frame is audio and
    /// nothing else — no per-frame header, no type tag. A second binary stream would be a
    /// protocol version decision, not a header field added quietly.
    private readonly record struct Outbound(byte[] Bytes, WebSocketMessageType Kind);

    /// How many audio frames may wait for one face before the oldest are dropped.
    ///
    /// Shallow on purpose. At the rate the voice produces them this is a fraction of a
    /// second, and a face that cannot keep up should hear a gap and catch up rather than
    /// fall further behind for the rest of the utterance. Old audio is worthless.
    private const int AudioQueueDepth = 16;

    /// One attached face, what it has asked not to be sent, and what it has asked *for*.
    ///
    /// A renderer wants visemes sixty times a second; a phone in a pocket wants a caption
    /// and the state of the house. Sending the first to the second is not merely wasteful,
    /// it is the difference between a usable mobile client and one that eats a battery over
    /// a mobile connection. `hello` already describes what the *host* can do; `subscribe` is
    /// the other half of that conversation.
    private sealed class Face(FaceId id, WebSocket socket)
    {
        /// Carried on the connection so the receive loop can say who a message came from.
        /// The id was always minted here; it simply never left this class.
        public FaceId Id { get; } = id;

        public WebSocket Socket { get; } = socket;

        /* One writer per face, fed by two queues, because sends used to be fired
           un-awaited — `_ = SendAsync(...)` — once per face per message.

           That is not the race it looks like: .NET serialises concurrent sends on one
           socket behind a lock rather than rejecting them, measured at 320 overlapping
           sends with nothing thrown. The real fault is what happens when a face stops
           reading. Its socket buffer fills, every queued send stops completing, and they
           pile up without bound holding their buffers alive — a slow phone becomes the
           host's memory leak. Ordering across un-awaited calls is not guaranteed either.

           A queue fixes all of it and makes back-pressure expressible, which is what
           audio needs. */

        /// Everything that is not audio. **Never dropped**: losing a `state` or a `caption`
        /// leaves a face permanently wrong about her rather than briefly behind.
        public Channel<Outbound> Control { get; } =
            Channel.CreateUnbounded<Outbound>(new UnboundedChannelOptions { SingleReader = true });

        /// Audio, bounded, oldest discarded first.
        public Channel<Outbound> Audio { get; } =
            Channel.CreateBounded<Outbound>(new BoundedChannelOptions(AudioQueueDepth)
            {
                SingleReader = true,
                FullMode = BoundedChannelFullMode.DropOldest
            });

        /// Replaced wholesale rather than mutated. `subscribe` arrives on this
        /// connection's receive thread while `Broadcast` reads this from whichever
        /// thread produced a viseme — and a HashSet being cleared and refilled under a
        /// concurrent `Contains` is a genuine race, not a theoretical one. Swapping a
        /// reference is atomic; editing a set is not.
        private volatile IReadOnlySet<string> _skip = new HashSet<string>();

        public IReadOnlySet<string> Skip
        {
            get => _skip;
            set => _skip = value;
        }

        /// The counterpart to `Skip`, and **opt-in** rather than opt-out.
        ///
        /// `skip` declines a rendering hint, so defaulting to "send it" is right: a new
        /// message type should reach an old face rather than be silently withheld. Audio is
        /// not a rendering hint, it is a *physical output* — and if it followed the same
        /// rule every browser face on this machine would start playing her voice on top of
        /// the speakers she is already using. That is not bandwidth, it is her talking over
        /// herself in the same room. A face that draws her mouth has not thereby claimed the
        /// right to make noise.
        private volatile IReadOnlySet<string> _want = new HashSet<string>();

        public IReadOnlySet<string> Want
        {
            get => _want;
            set => _want = value;
        }
    }

    private readonly ConcurrentDictionary<FaceId, Face> _faces = new();
    private readonly CancellationTokenSource _stopping = new();
    private TcpListener? _listener;
    private bool _disposed;

    public event Action<FaceMessage>? MessageReceived;

    public int Port { get; private set; }
    public string Token { get; } = Convert.ToHexString(RandomNumberGenerator.GetBytes(16)).ToLowerInvariant();
    public int FaceCount => _faces.Count;
    public bool IsRunning => _listener is not null;

    /// Whether a face on another machine may connect at all. Off means loopback only,
    /// which is what she has always been and remains the default.
    public bool Remote { get; private set; }

    /// Names this server in the log. The test harness starts a **real** server of this class
    /// on an ephemeral port with its own token, into the same log file — so "the last
    /// `face socket listening` line" used to hand back a dead test token for a port that no
    /// longer exists, and the failure presented as the *client* being broken. Reported from
    /// the Android side, where it cost real time. Anything but the app should say so.
    public string Label { get; init; } = "face socket";

    /// Returns false when the port could not be bound; the host then runs on the
    /// WebView2 channel alone rather than refusing to start.
    public bool Start(int port, bool remote = false)
    {
        Remote = remote;

        try
        {
            /* Loopback unless told otherwise. Binding every interface is what makes a
               phone possible and is also the single riskiest line in the project, which
               is why it is opt-in, logged loudly, and gated behind a separate key.

               It is still not a security boundary on its own: the intended deployment is
               Tailscale or Wireguard, where "every interface" means the tailnet and the
               LAN rather than the internet. A forwarded port would put a microphone and
               a house controller on the public internet behind one shared secret. */
            _listener = new TcpListener(remote ? IPAddress.Any : IPAddress.Loopback, port);
            _listener.Start();
            Port = ((IPEndPoint)_listener.LocalEndpoint).Port;
        }
        catch (Exception ex)
        {
            Log.Write($"{Label} could not bind port {port}: {ex.Message}");
            _listener = null;
            return false;
        }

        _ = Task.Run(AcceptLoop);
        Log.Write($"{Label} listening on ws://127.0.0.1:{Port}/?token={Token}");

        if (remote)
            Log.Warn($"face socket is ALSO listening on every interface, port {Port}. " +
                     "Remote faces must present the remote key. Reach it over Tailscale or " +
                     "Wireguard, never a forwarded port.");

        return true;
    }

    private async Task AcceptLoop()
    {
        while (!_stopping.IsCancellationRequested && _listener is not null)
        {
            TcpClient client;
            try
            {
                client = await _listener.AcceptTcpClientAsync(_stopping.Token);
            }
            catch (OperationCanceledException)
            {
                return;
            }
            catch (Exception ex)
            {
                if (!_stopping.IsCancellationRequested) Log.Write($"face socket accept failed: {ex.Message}");
                return;
            }

            _ = Task.Run(() => ServeAsync(client));
        }
    }

    private async Task ServeAsync(TcpClient client)
    {
        var id = FaceId.New();
        WebSocket? socket = null;

        try
        {
            client.NoDelay = true;
            var stream = client.GetStream();

            var request = await ReadRequestAsync(stream);
            if (request is null) { client.Close(); return; }

            var from = (client.Client.RemoteEndPoint as IPEndPoint)?.Address;

            /* A face's *sub-resources* cannot carry the query string. `<link href="face.css">`
               and `import('./watch.js')` are resolved by the browser, which knows nothing
               about a key, so gating them on `?key=` would serve the page and then refuse
               everything in it. The credential is therefore echoed back as a cookie on the
               way through, and the assets present that instead. */
            var credential = Credential(request.Value.Target, request.Value.Headers);

            if (!Authorised(credential, from))
            {
                Log.Write("face socket refused a connection with a bad or missing token");
                await WriteAsync(stream, "HTTP/1.1 401 Unauthorized\r\nConnection: close\r\n\r\n");
                client.Close();
                return;
            }

            // Not an upgrade: this is a browser asking for the page itself. Serving it is
            // what lets a renderer with no WebView2 virtual host — a phone — exist at all.
            if (!request.Value.Headers.TryGetValue("sec-websocket-key", out var key))
            {
                await ServeFileAsync(stream, request.Value, credential);
                client.Close();
                return;
            }

            var accept = Convert.ToBase64String(
                SHA1.HashData(Encoding.ASCII.GetBytes(key + HandshakeGuid)));

            await WriteAsync(stream,
                "HTTP/1.1 101 Switching Protocols\r\n" +
                "Upgrade: websocket\r\n" +
                "Connection: Upgrade\r\n" +
                $"Sec-WebSocket-Accept: {accept}\r\n\r\n");

            socket = WebSocket.CreateFromStream(stream, isServer: true,
                subProtocol: null, keepAliveInterval: TimeSpan.FromSeconds(30));

            var face = new Face(id, socket);
            _faces[id] = face;
            Log.Write($"face {id} connected over socket ({_faces.Count} attached)");

            // One writer per connection, for as long as the connection lasts.
            var pump = PumpAsync(face);

            await ReceiveLoopAsync(face);

            face.Control.Writer.TryComplete();
            face.Audio.Writer.TryComplete();
            await pump.WaitAsync(TimeSpan.FromSeconds(2)).ContinueWith(_ => { });
        }
        catch (Exception ex)
        {
            Log.Write($"face socket connection ended: {ex.Message}");
        }
        finally
        {
            _faces.TryRemove(id, out _);
            socket?.Dispose();
            client.Dispose();
        }
    }

    private async Task ReceiveLoopAsync(Face face)
    {
        var socket = face.Socket;
        var buffer = new byte[8192];
        var message = new MemoryStream();

        while (socket.State == WebSocketState.Open && !_stopping.IsCancellationRequested)
        {
            WebSocketReceiveResult result;
            try
            {
                result = await socket.ReceiveAsync(buffer, _stopping.Token);
            }
            catch (Exception)
            {
                return;
            }

            if (result.MessageType == WebSocketMessageType.Close)
            {
                // Complete the handshake. Dropping the socket instead leaves a face that
                // closed politely staring at an EOF error on its way out.
                try
                {
                    await socket.CloseOutputAsync(
                        WebSocketCloseStatus.NormalClosure, null, CancellationToken.None);
                }
                catch (Exception ex)
                {
                    Log.Write($"face socket close handshake: {ex.Message}");
                }
                return;
            }

            message.Write(buffer, 0, result.Count);
            if (!result.EndOfMessage) continue;

            var text = Encoding.UTF8.GetString(message.ToArray());
            message.SetLength(0);

            try
            {
                using var doc = JsonDocument.Parse(text);

                // `subscribe` is answered here rather than relayed: it is about this one
                // connection, and the session has no notion of which face asked.
                if (doc.RootElement.TryGetProperty("type", out var kind) &&
                    kind.GetString() == "subscribe")
                {
                    Subscribe(face, doc.RootElement);
                    continue;
                }

                MessageReceived?.Invoke(new FaceMessage(face.Id, doc.RootElement.Clone()));
            }
            catch (Exception ex)
            {
                Log.Write($"bad message from socket face: {ex.Message}");
            }
        }
    }

    /// `type` is passed in rather than parsed out: this is called for every viseme, and
    /// re-reading the JSON sixty times a second to learn what we just serialised would
    /// be work done purely to discard it.
    public void Broadcast(string json, string? type = null)
    {
        if (_faces.IsEmpty) return;

        byte[]? bytes = null;
        foreach (var (id, face) in _faces)
        {
            if (face.Socket.State != WebSocketState.Open) continue;
            if (type is not null && face.Skip.Contains(type)) continue;

            bytes ??= Encoding.UTF8.GetBytes(json);
            Enqueue(face, new Outbound(bytes, WebSocketMessageType.Text));
        }

        /* Anything but `speaking` means she has stopped, and buffered audio for a face is
           then a tail she has already finished — she would carry on talking on the phone
           after going quiet in the room. Dropping it here as well as in the face is the
           difference between a prompt stop and a nearly prompt one. */
        if (type == "state" && !json.Contains("\"speaking\"", StringComparison.Ordinal))
            foreach (var face in _faces.Values) DrainAudio(face);
    }

    /// Queue rather than send. Ordering is then guaranteed and a slow face cannot make the
    /// host hold buffers it will never manage to write.
    private static void Enqueue(Face face, Outbound item)
    {
        var queue = item.Kind == WebSocketMessageType.Binary ? face.Audio : face.Control;
        queue.Writer.TryWrite(item);
    }

    private static void DrainAudio(Face face)
    {
        while (face.Audio.Reader.TryRead(out _)) { }
    }

    /// One recipient, same rules. A face that asked to skip this type still skips it:
    /// being addressed directly is not a reason to override what it said it wanted, and
    /// a phone that opted out of visemes should not start receiving them because the
    /// host happened to know its id.
    public void SendTo(FaceId id, string json, string? type = null)
    {
        if (!_faces.TryGetValue(id, out var face)) return;
        if (face.Socket.State != WebSocketState.Open) return;
        if (type is not null && face.Skip.Contains(type)) return;

        Enqueue(face, new Outbound(Encoding.UTF8.GetBytes(json), WebSocketMessageType.Text));
    }

    /// Raw PCM to every face that asked for it, in the format `hello` advertised.
    ///
    /// **Opt-in, unlike everything else here.** See `Face.Want`: a face that has not asked
    /// receives nothing, because audio is a physical output rather than a rendering hint.
    public void BroadcastAudio(ReadOnlyMemory<byte> pcm)
    {
        if (_faces.IsEmpty) return;

        byte[]? bytes = null;
        foreach (var face in _faces.Values)
        {
            if (face.Socket.State != WebSocketState.Open) continue;
            if (!face.Want.Contains("audio")) continue;

            // Copied once, and only if somebody actually wants it — the caller's buffer is
            // pooled and goes back the moment this returns.
            bytes ??= pcm.ToArray();
            Enqueue(face, new Outbound(bytes, WebSocketMessageType.Binary));
        }
    }

    /// One writer per face, draining its queues in order. Control is always emptied before
    /// audio: it is rare, small, and being briefly behind on her voice is better than being
    /// wrong about her state.
    private async Task PumpAsync(Face face)
    {
        try
        {
            while (!_stopping.IsCancellationRequested && face.Socket.State == WebSocketState.Open)
            {
                while (face.Control.Reader.TryRead(out var control))
                    await face.Socket.SendAsync(control.Bytes, control.Kind, true, _stopping.Token);

                if (face.Audio.Reader.TryRead(out var audio))
                {
                    await face.Socket.SendAsync(audio.Bytes, audio.Kind, true, _stopping.Token);
                    continue;
                }

                var control_ = face.Control.Reader.WaitToReadAsync(_stopping.Token).AsTask();
                var audio_ = face.Audio.Reader.WaitToReadAsync(_stopping.Token).AsTask();
                await Task.WhenAny(control_, audio_);
            }
        }
        catch (OperationCanceledException)
        {
            // She is shutting down. Not a fault.
        }
        catch (Exception ex) when (ex is WebSocketException or ObjectDisposedException)
        {
            // A face that went away mid-send is routine and stays quiet — this is the case
            // the original catch-all was written for, and it was right about it.
        }
        catch (Exception ex)
        {
            // Anything else is a real fault, and it used to vanish. The catch-all removed
            // the face without a word, so a live renderer could be dropped and the only
            // symptom was that she went quiet, with nothing in the log to say why.
            Log.Error($"face {face.Id} send failed", ex);
        }
        finally
        {
                _faces.TryRemove(face.Id, out _);
        }
    }

    /// `{ "type": "subscribe", "skip": ["viseme", "level"] }`
    ///
    /// Opt-*out* rather than opt-in, deliberately: a face that says nothing keeps getting
    /// everything, so no existing renderer changes behaviour and a new message type
    /// reaches old clients rather than being silently withheld from them.
    private static void Subscribe(Face face, JsonElement message)
    {
        var wanted = new HashSet<string>();

        if (message.TryGetProperty("skip", out var skip) && skip.ValueKind == JsonValueKind.Array)
        {
            foreach (var entry in skip.EnumerateArray())
                if (entry.GetString() is { Length: > 0 } name) wanted.Add(name);
        }

        face.Skip = wanted;

        /* `want` is the other half, and it is opt-**in**. Absent means absent: no face
           receives audio until it asks. See Face.Want for why this one stream cannot
           follow the opt-out rule the rest of `subscribe` uses. */
        var asked = new HashSet<string>();

        if (message.TryGetProperty("want", out var want) && want.ValueKind == JsonValueKind.Array)
        {
            foreach (var entry in want.EnumerateArray())
                if (entry.GetString() is { Length: > 0 } name) asked.Add(name);
        }

        face.Want = asked;

        if (asked.Count > 0) Log.Write($"face {face.Id} asked for: {string.Join(", ", asked)}");

        Log.Write(face.Skip.Count == 0
            ? "face subscribed to everything"
            : $"face skipping: {string.Join(", ", face.Skip)}");
    }

    /// Who may connect, and with what.
    ///
    /// A connection from this machine may use the per-run token, which is what the
    /// built-in page and `attach-face.ps1` hold. **A connection from anywhere else may
    /// not** — that token is handed out in the log and in a URL, and it is scoped to a
    /// process on this box. A remote face presents the durable remote key instead, and
    /// only when remote access was switched on at start.
    /// What a request offered, from the query string if it has one and the cookie if it
    /// does not. The query wins: a fresh URL is a deliberate act, a cookie is a leftover.
    private static (string? Token, string? Key) Credential(string target, Dictionary<string, string> headers)
    {
        string? token = null, key = null;

        var query = target.IndexOf('?');
        if (query >= 0)
        {
            foreach (var pair in target[(query + 1)..].Split('&'))
            {
                var eq = pair.IndexOf('=');
                if (eq < 0) continue;

                var name = pair[..eq];
                var value = Uri.UnescapeDataString(pair[(eq + 1)..]);

                if (name.Equals("token", StringComparison.OrdinalIgnoreCase)) token = value;
                else if (name.Equals("key", StringComparison.OrdinalIgnoreCase)) key = value;
            }
        }

        if (token is not null || key is not null) return (token, key);

        if (!headers.TryGetValue("cookie", out var cookies)) return (null, null);

        foreach (var pair in cookies.Split(';'))
        {
            var eq = pair.IndexOf('=');
            if (eq < 0) continue;

            var name = pair[..eq].Trim();
            var value = pair[(eq + 1)..].Trim();

            if (name.Equals(CookieName, StringComparison.Ordinal)) token = value;
            else if (name.Equals(KeyCookieName, StringComparison.Ordinal)) key = value;
        }

        return (token, key);
    }

    private bool Authorised((string? Token, string? Key) offered, IPAddress? from)
    {
        var (token, key) = offered;
        var local = from is not null && IPAddress.IsLoopback(from);

        if (local)
        {
            // Fixed-time compare so the token cannot be guessed a character at a time.
            if (token is not null && CryptographicOperations.FixedTimeEquals(
                    Encoding.UTF8.GetBytes(token), Encoding.UTF8.GetBytes(Token)))
                return true;

            // A local client may also use the remote key, which is what makes it
            // testable from this machine without opening the socket to the network.
            return RemoteKey.Matches(key);
        }

        if (!Remote)
        {
            Log.Warn($"refused a face from {from}: remote access is off");
            return false;
        }

        if (RemoteKey.Matches(key))
        {
            Log.Write($"remote face authorised from {from}");
            return true;
        }

        Log.Warn($"refused a face from {from}: bad or missing remote key");
        return false;
    }

    private static async Task<(string Method, string Target, Dictionary<string, string> Headers)?> ReadRequestAsync(NetworkStream stream)
    {
        var head = new List<byte>(1024);
        var one = new byte[1];

        // Read only as far as the blank line. WebSocket.CreateFromStream must receive
        // the stream positioned exactly at the first frame, so nothing may be over-read.
        while (head.Count < 8192)
        {
            var read = await stream.ReadAsync(one);
            if (read == 0) return null;
            head.Add(one[0]);

            if (head.Count >= 4 &&
                head[^4] == '\r' && head[^3] == '\n' && head[^2] == '\r' && head[^1] == '\n') break;
        }

        var lines = Encoding.ASCII.GetString(head.ToArray()).Split("\r\n", StringSplitOptions.RemoveEmptyEntries);
        if (lines.Length == 0) return null;

        var parts = lines[0].Split(' ');
        if (parts.Length < 2) return null;

        var headers = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
        foreach (var line in lines.Skip(1))
        {
            var colon = line.IndexOf(':');
            if (colon > 0) headers[line[..colon].Trim()] = line[(colon + 1)..].Trim();
        }

        return (parts[0], parts[1], headers);
    }

    /// Answer a plain GET with a file from `wwwroot` or her avatars folder.
    ///
    /// The caller has already authorised this request, so the only questions left are
    /// whether the method is one we answer and whether the path names something we are
    /// willing to hand over. `StaticFiles` decides the second and says no by default.
    private async Task ServeFileAsync(NetworkStream stream, (string Method, string Target, Dictionary<string, string> Headers) request, (string? Token, string? Key) credential)
    {
        var head = request.Method.Equals("HEAD", StringComparison.OrdinalIgnoreCase);

        if (!head && !request.Method.Equals("GET", StringComparison.OrdinalIgnoreCase))
        {
            await WriteAsync(stream, "HTTP/1.1 405 Method Not Allowed\r\nAllow: GET, HEAD\r\nConnection: close\r\n\r\n");
            return;
        }

        if (StaticFiles.Resolve(request.Target) is not var (path, type))
        {
            await WriteAsync(stream, "HTTP/1.1 404 Not Found\r\nContent-Length: 0\r\nConnection: close\r\n\r\n");
            return;
        }

        byte[] body;
        try
        {
            body = await File.ReadAllBytesAsync(path, _stopping.Token);
        }
        catch (Exception ex)
        {
            Log.Write($"face http: could not read {path}: {ex.Message}");
            await WriteAsync(stream, "HTTP/1.1 500 Internal Server Error\r\nContent-Length: 0\r\nConnection: close\r\n\r\n");
            return;
        }

        /* Hand the credential back as a cookie so the page's own assets can be fetched.
           `SameSite=Strict` because no other site has any business causing a request
           here, and `Path=/` because `/avatars/` needs it as much as `/` does. It is not
           marked `Secure`: this socket speaks plain HTTP on a LAN behind Wireguard, and a
           `Secure` cookie would simply never be sent. */
        var cookies = "";
        if (credential.Token is { Length: > 0 } t)
            cookies += $"Set-Cookie: {CookieName}={t}; Path=/; HttpOnly; SameSite=Strict\r\n";
        if (credential.Key is { Length: > 0 } k)
            cookies += $"Set-Cookie: {KeyCookieName}={k}; Path=/; HttpOnly; SameSite=Strict\r\n";

        await WriteAsync(stream,
            "HTTP/1.1 200 OK\r\n" +
            $"Content-Type: {type}\r\n" +
            $"Content-Length: {body.Length}\r\n" +
            cookies +
            // Her face changes as often as she is rebuilt and is only ever served over a
            // LAN, so a cache that has to be reasoned about costs more than it saves.
            "Cache-Control: no-store\r\n" +
            "X-Content-Type-Options: nosniff\r\n" +
            "Connection: close\r\n\r\n");

        if (!head) await stream.WriteAsync(body, _stopping.Token);
    }

    private static Task WriteAsync(NetworkStream stream, string text) =>
        stream.WriteAsync(Encoding.ASCII.GetBytes(text)).AsTask();

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        _stopping.Cancel();
        try { _listener?.Stop(); } catch (Exception ex) { Log.Write($"face socket stop: {ex.Message}"); }

        foreach (var face in _faces.Values) face.Socket.Dispose();
        _faces.Clear();
        _stopping.Dispose();
    }
}
```
