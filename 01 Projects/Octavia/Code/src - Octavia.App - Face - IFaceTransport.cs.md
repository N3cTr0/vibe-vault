---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\IFaceTransport.cs
---

# src\Octavia.App\Face\IFaceTransport.cs

```csharp
using System.Text.Json;

namespace Octavia.Face;

/// The face is a renderer and nothing more. Today it is a WebView2 in this process;
/// keeping the transport behind an interface is what lets it become a socket later,
/// with the face on a tablet on the wall.
internal interface IFaceTransport
{
    event Action<FaceMessage>? MessageReceived;

    /// `to` is optional and null keeps the original meaning: **everyone**.
    ///
    /// That default is the reason this change stayed small — nearly every send site is
    /// untouched — and it is the same instinct as `subscribe` being opt-*out*: a new
    /// message type reaches every face rather than being silently withheld from the ones
    /// nobody remembered to address.
    void Send(object message, FaceId? to = null);

    /// The renderer that is always there — the built-in page — or null when there is
    /// none. Somewhere to send a message that must reach exactly one face when nothing
    /// else identifies which. Deliberately separate from `FaceStatus`, which is a
    /// diagnostics answer and should stay one.
    FaceId? BuiltInFace { get; }

    /// What is actually attached right now. Only a transport knows, and "is anything
    /// listening?" is the first question a diagnostics report has to answer — while the
    /// session still learns nothing about how any particular face connected.
    FaceStatus Status { get; }
}

/// <param name="Page">The built-in WebView2 page exists.</param>
/// <param name="SocketBound">The loopback listener started.</param>
/// <param name="Port">Its port, or 0 if it never bound.</param>
/// <param name="SocketFaces">How many faces are attached over the socket.</param>
internal readonly record struct FaceStatus(bool Page, bool SocketBound, int Port, int SocketFaces)
{
    public int Attached => (Page ? 1 : 0) + SocketFaces;
}
```
