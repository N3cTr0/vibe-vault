---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Being.cs
---

# src\Octavia.Core\Being.cs

```csharp
using Octavia.Core;
using Octavia.Face;

namespace Octavia;

/// Her, with nothing to look at: the config, the socket, the hub and the session.
///
/// **This is the whole of what a server is.** Everything a renderer needs already travels
/// over the socket, so a host with no window is not a reduced Octavia — it is the same one
/// with `FaceHub(page: null, ...)`. That the constructor already took a nullable page is
/// not luck; it is Stage 3 having been done properly, and this class is the first caller to
/// use the null.
///
/// The window keeps its own wiring rather than going through here, because it has a second
/// transport to offer and a WebView2 to build first. See `MainWindow`.
internal sealed class Being : IDisposable
{
    private Being(OctaviaConfig config, WebSocketFaceServer sockets, FaceHub hub, OctaviaSession session)
    {
        Config = config;
        Sockets = sockets;
        Hub = hub;
        Session = session;
        Ears = session.WarmEarsAsync();
    }

    public OctaviaConfig Config { get; }
    public WebSocketFaceServer Sockets { get; }
    public FaceHub Hub { get; }
    public OctaviaSession Session { get; }

    /// Completes with the engine's name once the speech models are loaded, or with null when
    /// `OpenEarsOnStart` is off or they would not open.
    ///
    /// **Not awaited before the socket is serving.** A face that attaches during the load
    /// gets a working Octavia who cannot hear yet, which is a far better first impression
    /// than a server that appears not to have started — and `hello` carries `ears` and is
    /// re-sent when they open, so the placard corrects itself with nobody reloading.
    public Task<string?> Ears { get; }

    /// Where a face should point itself, loopback-shaped. A remote face uses the remote key
    /// against this machine's address instead; see `RemoteKey`.
    public string Address => $"http://127.0.0.1:{Sockets.Port}/index.html?token={Sockets.Token}";

    /// Null when the port could not be bound.
    ///
    /// **Fatal here, where it is survivable in the window.** A window that loses the socket
    /// still has the built-in page on its own channel and is worth starting; a server that
    /// loses it has no way for anything to reach her at all, and pretending otherwise would
    /// leave a process running that looks alive and answers nobody.
    public static Being? Start(OctaviaConfig config)
    {
        /* Before anything reads a tool server's settings: anything named like a key or a
           password moves out of `config.json` and into the sealed store. See
           `SecretStore.SealLoose` — it is a no-op after the first run, and deliberately does
           nothing at all when this process is LocalSystem. */
        if (SecretStore.SealLoose(config) is { Count: > 0 } moved)
            Log.Write($"sealed and removed from config.json: {string.Join(", ", moved)}");

        /* The dashboard's origin, not its URL: a `frame-src` naming a whole path would look
           more precise than CSP actually is, and an unparseable one widens nothing rather
           than widening everything. */
        StaticFiles.FrameOrigin =
            Uri.TryCreate(config.DashboardUrl, UriKind.Absolute, out var dash)
                ? dash.GetLeftPart(UriPartial.Authority)
                : "";

        if (config.DashboardUrl.Length > 0 && StaticFiles.FrameOrigin.Length == 0)
            Log.Warn($"dashboard '{config.DashboardUrl}' is not an absolute URL; the button will not open it");

        var sockets = new WebSocketFaceServer();

        if (!sockets.Start(config.FacePort, config.RemoteAccess))
        {
            sockets.Dispose();
            return null;
        }

        var hub = new FaceHub(sockets);
        return new Being(config, sockets, hub, new OctaviaSession(config, hub));
    }

    public void Dispose()
    {
        Session.Dispose();
        Hub.Dispose();
        Sockets.Dispose();
    }
}
```
