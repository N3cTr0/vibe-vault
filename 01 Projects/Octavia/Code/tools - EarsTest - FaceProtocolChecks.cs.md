---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\FaceProtocolChecks.cs
---

# tools\EarsTest\FaceProtocolChecks.cs

```csharp
using System.Net.WebSockets;
using System.Text;
using System.Text.Json;
using Octavia.Face;

/// Exercises the face protocol server in-process: token enforcement, both directions,
/// and fan-out to several faces at once. No running app required.
internal static class FaceProtocolChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;

        void Check(string name, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        using var server = new WebSocketFaceServer();

        // Port 0 asks the OS for a free one, so the test never fights the running app.
        if (!server.Start(0)) { Console.WriteLine("  FAIL could not start server"); return 1; }
        Check("server started", server.Port > 0, $"port {server.Port}");

        var inbound = new List<JsonElement>();
        server.MessageReceived += m => { lock (inbound) inbound.Add(m); };

        // --- a connection without the token must never become a WebSocket -----------
        try
        {
            using var intruder = new ClientWebSocket();
            using var giveUp = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            await intruder.ConnectAsync(FaceUri(server.Port, "wrong-token"), giveUp.Token);
            Check("bad token refused", false, "it was accepted");
        }
        catch (Exception)
        {
            Check("bad token refused", true);
        }

        try
        {
            using var anonymous = new ClientWebSocket();
            using var giveUp = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            await anonymous.ConnectAsync(new Uri($"ws://127.0.0.1:{server.Port}/"), giveUp.Token);
            Check("missing token refused", false, "it was accepted");
        }
        catch (Exception)
        {
            Check("missing token refused", true);
        }

        // --- two legitimate faces ---------------------------------------------------
        using var faceA = new ClientWebSocket();
        using var faceB = new ClientWebSocket();
        await faceA.ConnectAsync(FaceUri(server.Port, server.Token), Cancel());
        await faceB.ConnectAsync(FaceUri(server.Port, server.Token), Cancel());
        Check("two faces attached", server.FaceCount == 2, $"{server.FaceCount} attached");

        // --- host to face, fanned out ----------------------------------------------
        server.Broadcast("""{"type":"state","value":"listening"}""");

        var fromA = await ReadAsync(faceA);
        var fromB = await ReadAsync(faceB);
        Check("both faces received the broadcast",
            fromA.Contains("listening") && fromB.Contains("listening"),
            fromA);

        // --- face to host -----------------------------------------------------------
        await SendAsync(faceB, """{"type":"say","text":"hello from a second face"}""");

        var arrived = await Wait(() => { lock (inbound) return inbound.Count > 0; });
        Check("host received the face message", arrived);

        if (arrived)
        {
            JsonElement message;
            lock (inbound) message = inbound[0];
            Check("message parsed with its fields",
                message.GetProperty("type").GetString() == "say" &&
                message.GetProperty("text").GetString() == "hello from a second face");
        }

        // --- a face that leaves is dropped ------------------------------------------
        await faceB.CloseAsync(WebSocketCloseStatus.NormalClosure, null, Cancel());
        var dropped = await Wait(() => server.FaceCount == 1);
        Check("departed face dropped", dropped, $"{server.FaceCount} still attached");

        return failures;
    }

    private static Uri FaceUri(int port, string token) =>
        new($"ws://127.0.0.1:{port}/?token={Uri.EscapeDataString(token)}");

    private static CancellationToken Cancel() =>
        new CancellationTokenSource(TimeSpan.FromSeconds(10)).Token;

    private static async Task SendAsync(ClientWebSocket socket, string json) =>
        await socket.SendAsync(Encoding.UTF8.GetBytes(json),
            WebSocketMessageType.Text, true, Cancel());

    private static async Task<string> ReadAsync(ClientWebSocket socket)
    {
        var buffer = new byte[8192];
        var result = await socket.ReceiveAsync(buffer, Cancel());
        return Encoding.UTF8.GetString(buffer, 0, result.Count);
    }

    private static async Task<bool> Wait(Func<bool> until, int millis = 3000)
    {
        var deadline = DateTime.UtcNow.AddMilliseconds(millis);
        while (DateTime.UtcNow < deadline)
        {
            if (until()) return true;
            await Task.Delay(25);
        }
        return until();
    }
}
```
