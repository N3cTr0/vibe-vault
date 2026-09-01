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

        // Labelled, because this is a *real* server writing to her real log. An unlabelled
        // line here is indistinguishable from the app's, and handing someone a dead test
        // token for a vanished port presents as the client being broken.
        using var server = new WebSocketFaceServer { Label = "TEST face socket" };

        // Port 0 asks the OS for a free one, so the test never fights the running app.
        if (!server.Start(0)) { Console.WriteLine("  FAIL could not start server"); return 1; }
        Check("server started", server.Port > 0, $"port {server.Port}");

        var inbound = new List<FaceMessage>();
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

        /* --- the remote key, through the real socket --------------------------------

           A loopback client may present the remote key instead of the token, and that is
           there precisely so this path can be walked without a second machine. Nothing
           walked it until v0.23.1, and it had been broken since v0.14.0: a key was minted
           fresh on every read, so the offered one was compared against a secret that had
           existed for a microsecond.

           `RemoteKeyChecks` proves the key round-trips. This proves the *server* accepts
           one, which is the part a phone actually depends on. Against a scratch file, so
           running the checks never unpairs a real device. */
        var savedKeyPath = Environment.GetEnvironmentVariable("OCTAVIA_REMOTE_KEY");
        var scratchKey = Path.Combine(Path.GetTempPath(), $"octavia-proto-{Guid.NewGuid():N}.key");
        Environment.SetEnvironmentVariable("OCTAVIA_REMOTE_KEY", scratchKey);

        try
        {
            var key = RemoteKey.Value;

            using var byKey = new ClientWebSocket();
            using var giveUp = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            await byKey.ConnectAsync(new Uri($"ws://127.0.0.1:{server.Port}/?key={key}"), giveUp.Token);
            Check("the remote key opens a socket", byKey.State == WebSocketState.Open, byKey.State.ToString());
            await byKey.CloseAsync(WebSocketCloseStatus.NormalClosure, "done", Cancel());
        }
        catch (Exception ex)
        {
            Check("the remote key opens a socket", false, ex.Message);
        }

        try
        {
            using var wrong = new ClientWebSocket();
            using var giveUp = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            await wrong.ConnectAsync(new Uri($"ws://127.0.0.1:{server.Port}/?key=ABCDE-FGHJK-MNPQR-STUVW"), giveUp.Token);
            Check("a wrong remote key is refused", false, "it was accepted");
        }
        catch (Exception)
        {
            Check("a wrong remote key is refused", true);
        }
        finally
        {
            Environment.SetEnvironmentVariable("OCTAVIA_REMOTE_KEY", savedKeyPath);
            try { File.Delete(scratchKey); } catch { /* a temp file that outlives the run is harmless */ }
        }

        // --- two legitimate faces ---------------------------------------------------
        using var faceA = new ClientWebSocket();
        using var faceB = new ClientWebSocket();
        await faceA.ConnectAsync(FaceUri(server.Port, server.Token), Cancel());
        await faceB.ConnectAsync(FaceUri(server.Port, server.Token), Cancel());

        /* Waited for, not asserted outright. `ConnectAsync` returns when the *client* has
           the 101 response, which is strictly before the server has finished building the
           face and putting it in the table — so reading the count on the next line was
           always a race the test happened to win.

           It started losing every time when the face gained its send queues: two channel
           allocations between writing the handshake and the insert was enough. The
           widened window exposed the flake rather than creating it. */
        Check("two faces attached", await Wait(() => server.FaceCount == 2),
            $"{server.FaceCount} attached");

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
            FaceMessage received;
            lock (inbound) received = inbound[0];
            var message = received.Body;
            Check("message parsed with its fields",
                message.GetProperty("type").GetString() == "say" &&
                message.GetProperty("text").GetString() == "hello from a second face");

            // Stage 14 item 1: the host can tell who spoke. Without this the session
            // cannot address one face, which is what made `look` open every camera.
            Check("the message carries its sender", received.From != default,
                received.From == default ? "FaceId was default" : $"from {received.From}");
        }

        /* --- addressed to one face, not all of them --------------------------------
           The point of Stage 14 item 1. `look` used to be broadcast, so a tablet and a
           desktop both opened their cameras for one question and the first frame back
           won arbitrarily. This proves the transport can now reach exactly one.

           faceB is the sender above, so it is the one the host knows about — the same
           way the session learns which face a person is speaking through. */
        if (arrived)
        {
            FaceId knownFace;
            lock (inbound) knownFace = inbound[0].From;

            server.SendTo(knownFace, """{"type":"look"}""");

            var reachedB = await ReadAsync(faceB);
            Check("a targeted send reaches the face it named", reachedB.Contains("look"), reachedB);

            // The other face must hear nothing at all. Read with a short timeout: the
            // assertion is silence, so waiting for a message that should never come is
            // the only way to check it.
            var reachedA = await ReadAsync(faceA, TimeSpan.FromSeconds(1));
            Check("the other face hears nothing", reachedA is null, reachedA ?? "");
        }

        /* --- a burst must not drop a healthy face -----------------------------------
           Broadcast used to fire `_ = SendAsync(...)` per face, un-awaited.

           **Not** for the reason first suspected: .NET serialises concurrent sends on one
           socket behind a lock rather than throwing — measured directly at 320 overlapping
           sends with nothing thrown and the socket still open. The real fault is a face
           that stops reading: its buffer fills, the queued sends stop completing, and they
           accumulate without bound holding their buffers alive.

           This check guards the fan-out under pressure. Criterion 7 — a face that never
           reads at all — is the memory half and is not cheap to assert here. */
        var before = server.FaceCount;
        var payload = new string('x', 32000);

        // Neither client is reading, so the socket buffers fill and a send stops completing
        // instantly — which is the only condition under which the un-awaited calls overlap.
        // Fired from several threads as well, because that is the real shape: visemes come
        // off the audio path while captions come off the turn.
        Parallel.For(0, 8, _ =>
        {
            for (var i = 0; i < 60; i++)
                server.Broadcast($$"""{"type":"caption","text":"{{payload}}"}""");
        });

        await Task.Delay(1500);
        Check("a burst of sends does not drop a healthy face",
            server.FaceCount == before, $"{before} before, {server.FaceCount} after");

        /* --- audio is opt-IN, and this is the check that matters most ----------------
           Acceptance criterion 1 of Stage 14 item 3. If audio followed the opt-*out* rule
           the rest of `subscribe` uses, every browser face on this machine would start
           playing her voice on top of the speakers she is already using — her talking over
           herself in the same room. A face that draws her mouth has not claimed the right
           to make noise. */
        await SendAsync(faceB, """{"type":"subscribe","want":["audio"]}""");
        await Task.Delay(300);

        var pcm = new byte[640];
        for (var i = 0; i < pcm.Length; i++) pcm[i] = (byte)i;

        FaceId[] everyone;
        lock (inbound) everyone = [inbound[0].From];

        server.SendAudioTo(pcm, everyone);

        var audioB = await ReadBinaryAsync(faceB, TimeSpan.FromSeconds(2));
        Check("a face that asked for audio receives it",
            audioB == pcm.Length, $"got {audioB?.ToString() ?? "nothing"} bytes");

        var audioA = await ReadBinaryAsync(faceA, TimeSpan.FromSeconds(1));
        Check("a face that did NOT ask receives no audio at all",
            audioA is null, $"it received {audioA} bytes");

        /* --- her voice does not leak into the next room ------------------------------
           Stage 14 item 9, acceptance criterion 3. `SendAudio` took no recipients at all
           until now, so every face that had opted in heard her — she answered a question
           asked on a handset out loud at an empty desk. Addressing a face that wants audio
           and asserting the *other* one stays silent is the transport half of that. */
        server.SendAudioTo(pcm, []);
        var toNobody = await ReadBinaryAsync(faceB, TimeSpan.FromSeconds(1));
        Check("audio addressed to no room reaches nobody", toNobody is null,
            $"a face received {toNobody} bytes anyway");

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

    /// Returns null if nothing arrived inside `within`. Needed to assert a *silence* —
    /// that a face which was not addressed received nothing — which cannot be checked any
    /// other way than by waiting for a message that should never come.
    ///
    /// **The read is raced against a delay rather than cancelled.** Cancelling a
    /// `ReceiveAsync` *aborts* a WebSocket, it does not time out — so the obvious version
    /// of this helper killed the very face it was proving had been left alone, and the
    /// next check then reported it as departed. That lesson was already written down in
    /// this project and still caught me. The receive is simply left pending; the socket
    /// is disposed at the end of the test either way.
    private static async Task<string?> ReadAsync(ClientWebSocket socket, TimeSpan within)
    {
        var buffer = new byte[8192];
        var reading = socket.ReceiveAsync(buffer, CancellationToken.None);

        if (await Task.WhenAny(reading, Task.Delay(within)) != reading) return null;

        var result = await reading;
        return Encoding.UTF8.GetString(buffer, 0, result.Count);
    }

    /// Byte count of the next **binary** frame, or null if nothing arrived in time.
    ///
    /// Text frames are skipped rather than counted: a caption arriving first must not be
    /// mistaken for audio, and — more to the point — must not be mistaken for its absence.
    /// Raced against a delay, never cancelled; cancelling a receive aborts the socket.
    private static async Task<int?> ReadBinaryAsync(ClientWebSocket socket, TimeSpan within)
    {
        var deadline = DateTime.UtcNow + within;

        while (DateTime.UtcNow < deadline)
        {
            var buffer = new byte[16 * 1024];
            var reading = socket.ReceiveAsync(buffer, CancellationToken.None);

            if (await Task.WhenAny(reading, Task.Delay(deadline - DateTime.UtcNow)) != reading)
                return null;

            var result = await reading;
            if (result.MessageType == WebSocketMessageType.Binary) return result.Count;
        }

        return null;
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
