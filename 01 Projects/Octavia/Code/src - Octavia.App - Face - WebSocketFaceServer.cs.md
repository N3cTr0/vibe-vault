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

    /// One attached face, and what it has asked not to be sent.
    ///
    /// A renderer wants visemes sixty times a second; a phone in a pocket wants a
    /// caption and the state of the house. Sending the first to the second is not merely
    /// wasteful, it is the difference between a usable mobile client and one that eats a
    /// battery over a mobile connection. `hello` already describes what the *host* can
    /// do; `subscribe` is the other half of that conversation.
    private sealed class Face(FaceId id, WebSocket socket)
    {
        /// Carried on the connection so the receive loop can say who a message came from.
        /// The id was always minted here; it simply never left this class.
        public FaceId Id { get; } = id;

        public WebSocket Socket { get; } = socket;

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
            Log.Write($"face socket could not bind port {port}: {ex.Message}");
            _listener = null;
            return false;
        }

        _ = Task.Run(AcceptLoop);
        Log.Write($"face socket listening on ws://127.0.0.1:{Port}/?token={Token}");

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

            await ReceiveLoopAsync(face);
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
            _ = SendAsync(id, face.Socket, bytes);
        }
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

        _ = SendAsync(id, face.Socket, Encoding.UTF8.GetBytes(json));
    }

    private async Task SendAsync(FaceId id, WebSocket socket, byte[] bytes)
    {
        try
        {
            await socket.SendAsync(bytes, WebSocketMessageType.Text, true, _stopping.Token);
        }
        catch (Exception)
        {
            // A face that went away mid-send is not an error worth logging every frame;
            // viseme messages would flood the log.
            _faces.TryRemove(id, out _);
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
