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
    event Action<JsonElement>? MessageReceived;
    void Send(object message);

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
