---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\PageHost.cs
---

# tools\EarsTest\PageHost.cs

```csharp
using Octavia.Face;

/// A real socket server for the checks that drive the real page.
///
/// **Why this exists, and why it did not before.** Until Stage 15 the page had a second
/// transport: `postMessage` to the WebView2 host in her own process. That was the cheapest
/// way for a check to pretend to be the host — `PostWebMessageAsJson` a `hello`, read what
/// came back off `WebMessageReceived` — and both `SyntaxChecks` and `EmbedderChecks` used
/// it. Stage 15 deleted that channel, because after the split there is nothing at the other
/// end of it.
///
/// So the checks speak the protocol the page actually speaks. That is a strictly better
/// test and it caught something the old one could not: the page now announces `ready` from
/// the socket's `open` handler, so a page with no socket never announces at all — which is
/// correct, and was invisible while a check could bypass the socket entirely.
internal sealed class PageHost : IDisposable
{
    private readonly WebSocketFaceServer _server;
    private readonly List<string> _fromFace = [];

    public PageHost()
    {
        // Named, because the harness and the app write to the same log and a check's token
        // being mistaken for the app's has cost real time before. See `Label`.
        _server = new WebSocketFaceServer { Label = "check face socket" };

        if (!_server.Start(0))
            throw new InvalidOperationException("the check's face socket would not bind");

        _server.MessageReceived += message =>
        {
            lock (_fromFace) _fromFace.Add(message.Body.GetRawText());
        };
    }

    public int Port => _server.Port;
    public string Token => _server.Token;

    /// The page, served by this server over loopback.
    ///
    /// **Loopback HTTP is a secure context**, so this is the desktop client's real
    /// arrangement rather than an approximation of it: same origin for the page and the
    /// socket, `getUserMedia` available, exactly what `Octavia.App` now loads.
    public string Url => $"http://127.0.0.1:{Port}/index.html?token={Token}";

    /// The socket, for a page served from somewhere else — a virtual host, so that a check
    /// can reproduce an *insecure* origin, which loopback can never be.
    public string Query => $"?port={Port}&token={Token}";

    public void Send(string json, string? type = null) => _server.Broadcast(json, type);

    /// Everything the face has said, as one string. Crude on purpose: these checks ask
    /// whether a message was sent at all, and a substring answers that without teaching the
    /// harness the shape of every message.
    public string Heard()
    {
        lock (_fromFace) return string.Join(" ", _fromFace);
    }

    public void Clear()
    {
        lock (_fromFace) _fromFace.Clear();
    }

    /// Waits for the face to say something of this kind, or gives up.
    ///
    /// Polled rather than slept against: a socket handshake plus a scene build is tens of
    /// milliseconds on a good day and seconds on a cold one, and a fixed delay long enough
    /// for the second is wasted on every run of the first.
    public bool Wait(string type, TimeSpan within)
    {
        var deadline = DateTime.UtcNow + within;

        while (DateTime.UtcNow < deadline)
        {
            if (Heard().Contains($"\"type\":\"{type}\"")) return true;
            System.Windows.Forms.Application.DoEvents();
            Thread.Sleep(25);
        }

        return false;
    }

    public void Dispose() => _server.Dispose();
}
```
