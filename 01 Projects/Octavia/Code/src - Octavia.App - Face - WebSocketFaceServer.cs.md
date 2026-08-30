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

    private readonly ConcurrentDictionary<Guid, WebSocket> _faces = new();
    private readonly CancellationTokenSource _stopping = new();
    private TcpListener? _listener;
    private bool _disposed;

    public event Action<JsonElement>? MessageReceived;

    public int Port { get; private set; }
    public string Token { get; } = Convert.ToHexString(RandomNumberGenerator.GetBytes(16)).ToLowerInvariant();
    public int FaceCount => _faces.Count;
    public bool IsRunning => _listener is not null;

    /// Returns false when the port could not be bound; the host then runs on the
    /// WebView2 channel alone rather than refusing to start.
    public bool Start(int port)
    {
        try
        {
            // Loopback only. Binding 0.0.0.0 would expose her to the network.
            _listener = new TcpListener(IPAddress.Loopback, port);
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
        var id = Guid.NewGuid();
        WebSocket? socket = null;

        try
        {
            client.NoDelay = true;
            var stream = client.GetStream();

            var request = await ReadRequestAsync(stream);
            if (request is null) { client.Close(); return; }

            if (!Authorised(request.Value.Target))
            {
                Log.Write("face socket refused a connection with a bad or missing token");
                await WriteAsync(stream, "HTTP/1.1 401 Unauthorized\r\nConnection: close\r\n\r\n");
                client.Close();
                return;
            }

            if (!request.Value.Headers.TryGetValue("sec-websocket-key", out var key))
            {
                await WriteAsync(stream, "HTTP/1.1 400 Bad Request\r\nConnection: close\r\n\r\n");
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

            _faces[id] = socket;
            Log.Write($"face connected over socket ({_faces.Count} attached)");

            await ReceiveLoopAsync(socket);
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

    private async Task ReceiveLoopAsync(WebSocket socket)
    {
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
                MessageReceived?.Invoke(doc.RootElement.Clone());
            }
            catch (Exception ex)
            {
                Log.Write($"bad message from socket face: {ex.Message}");
            }
        }
    }

    public void Broadcast(string json)
    {
        if (_faces.IsEmpty) return;

        var bytes = Encoding.UTF8.GetBytes(json);
        foreach (var (id, socket) in _faces)
        {
            if (socket.State != WebSocketState.Open) continue;
            _ = SendAsync(id, socket, bytes);
        }
    }

    private async Task SendAsync(Guid id, WebSocket socket, byte[] bytes)
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

    private bool Authorised(string target)
    {
        var query = target.IndexOf('?');
        if (query < 0) return false;

        foreach (var pair in target[(query + 1)..].Split('&'))
        {
            var eq = pair.IndexOf('=');
            if (eq < 0) continue;
            if (!pair[..eq].Equals("token", StringComparison.OrdinalIgnoreCase)) continue;

            // Fixed-time compare so the token cannot be guessed a character at a time.
            return CryptographicOperations.FixedTimeEquals(
                Encoding.UTF8.GetBytes(Uri.UnescapeDataString(pair[(eq + 1)..])),
                Encoding.UTF8.GetBytes(Token));
        }

        return false;
    }

    private static async Task<(string Target, Dictionary<string, string> Headers)?> ReadRequestAsync(NetworkStream stream)
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

        return (parts[1], headers);
    }

    private static Task WriteAsync(NetworkStream stream, string text) =>
        stream.WriteAsync(Encoding.ASCII.GetBytes(text)).AsTask();

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        _stopping.Cancel();
        try { _listener?.Stop(); } catch (Exception ex) { Log.Write($"face socket stop: {ex.Message}"); }

        foreach (var socket in _faces.Values) socket.Dispose();
        _faces.Clear();
        _stopping.Dispose();
    }
}
```
