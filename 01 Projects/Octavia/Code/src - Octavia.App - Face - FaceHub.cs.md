---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\FaceHub.cs
---

# src\Octavia.App\Face\FaceHub.cs

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Octavia.Face;

/// Fans one message out to every attached face and merges what comes back.
///
/// The built-in page and any socket faces are peers: the session pushes a message here
/// and does not know or care how many renderers are listening, or which transport each
/// one chose. That is the whole point of the stage — the being stops knowing about the
/// renderer.
internal sealed class FaceHub : IFaceTransport, IDisposable
{
    public static readonly JsonSerializerOptions Json = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    private readonly WebViewFaceTransport? _page;
    private readonly WebSocketFaceServer? _sockets;

    public event Action<FaceMessage>? MessageReceived;

    public FaceHub(WebViewFaceTransport? page, WebSocketFaceServer? sockets)
    {
        _page = page;
        _sockets = sockets;

        if (_page is not null) _page.MessageReceived += Relay;
        if (_sockets is not null) _sockets.MessageReceived += Relay;
    }

    public FaceId? BuiltInFace => _page?.Id;

    public FaceStatus Status => new(
        Page: _page is not null,
        SocketBound: _sockets?.IsRunning ?? false,
        Port: _sockets?.Port ?? 0,
        SocketFaces: _sockets?.FaceCount ?? 0);

    private void Relay(FaceMessage message) => MessageReceived?.Invoke(message);

    /// `to` is null for all but a handful of messages, and null means everyone — the
    /// behaviour this had before faces had identity. Only something that belongs to one
    /// renderer, like `look`, names a recipient.
    public void Send(object message, FaceId? to = null)
    {
        // Serialise once, however many faces are attached.
        var json = JsonSerializer.Serialize(message, Json);

        // The type is read off the object rather than back out of the JSON, so a face
        // that has opted out of visemes costs nothing to skip.
        var type = message.GetType().GetProperty("type")?.GetValue(message) as string;

        if (to is null)
        {
            _page?.SendJson(json);
            _sockets?.Broadcast(json, type);
            return;
        }

        if (_page is not null && to == _page.Id)
        {
            _page.SendJson(json);
            return;
        }

        _sockets?.SendTo(to.Value, json, type);
    }

    public void Dispose()
    {
        if (_page is not null) _page.MessageReceived -= Relay;
        if (_sockets is not null) _sockets.MessageReceived -= Relay;
    }
}
```
