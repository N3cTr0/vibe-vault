---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Face\FaceHub.cs
---

# src\Octavia.Core\Face\FaceHub.cs

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Octavia.Face;

/// Fans one message out to every attached face and merges what comes back.
///
/// The session pushes a message here and does not know or care how many renderers are
/// listening, or which transport each one chose. That is the whole point of Stage 3 — the
/// being stops knowing about the renderer.
///
/// **It used to merge two transports and now adapts one.** The other was
/// `WebViewFaceTransport`, a `postMessage` channel to the page hosted in her own process;
/// Stage 15 moved the session out of that process, so every face — including the desktop
/// client's — is a socket face now.
///
/// It is kept rather than collapsed into the socket server, because *merging transports is
/// what this class is for* and a second one is already specified: item 3 has the client
/// lending the server its microphone and speakers, which arrives here as a peer of the
/// socket and not as a special case inside it.
internal sealed class FaceHub : IFaceTransport, IDisposable
{
    public static readonly JsonSerializerOptions Json = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    private readonly WebSocketFaceServer? _sockets;

    public event Action<FaceMessage>? MessageReceived;

    public FaceHub(WebSocketFaceServer? sockets)
    {
        _sockets = sockets;

        if (_sockets is null) return;

        _sockets.MessageReceived += Relay;
        _sockets.AudioReceived += (id, pcm) => AudioReceived?.Invoke(id, pcm);
        _sockets.FaceDeparted += id => FaceDeparted?.Invoke(id);
    }

    public void SendAudio(ReadOnlyMemory<byte> pcm, IReadOnlyCollection<FaceId> to) =>
        _sockets?.SendAudioTo(pcm, to);

    public bool AnyWantsAudio(IReadOnlyCollection<FaceId> faces) =>
        _sockets?.AnyWantsAudio(faces) ?? false;

    /// **Always null now, and honestly so.** It meant "the renderer that is always there",
    /// and with a server there is no such thing: every face comes and goes over a socket.
    /// `EyesIn` used it only as a tie-break for which face to ask for a still, and falls
    /// back to the first face that claims a camera — which is the same answer in every room
    /// that has one camera in it, and every room does.
    public FaceId? BuiltInFace => null;

    public FaceStatus Status => new(
        Page: false,
        SocketBound: _sockets?.IsRunning ?? false,
        Port: _sockets?.Port ?? 0,
        SocketFaces: _sockets?.FaceCount ?? 0);

    public event Action<FaceId, byte[]>? AudioReceived;
    public event Action<FaceId>? FaceDeparted;

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

        if (to is null) _sockets?.Broadcast(json, type);
        else _sockets?.SendTo(to.Value, json, type);
    }

    /// Serialised once, delivered to each named face. An empty list sends nothing at all,
    /// which is the right answer and not a mistake: a room with nobody in it is a room with
    /// nobody in it, and falling back to "everyone" here is exactly the bug rooms exist to
    /// fix.
    public void SendMany(object message, IReadOnlyCollection<FaceId> to)
    {
        if (to.Count == 0) return;

        var json = JsonSerializer.Serialize(message, Json);
        var type = message.GetType().GetProperty("type")?.GetValue(message) as string;

        foreach (var face in to) _sockets?.SendTo(face, json, type);
    }

    public void Dispose()
    {
        if (_sockets is not null) _sockets.MessageReceived -= Relay;
    }
}
```
