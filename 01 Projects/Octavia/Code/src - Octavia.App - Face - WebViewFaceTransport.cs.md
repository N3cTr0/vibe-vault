---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\WebViewFaceTransport.cs
---

# src\Octavia.App\Face\WebViewFaceTransport.cs

```csharp
using System.Text.Json;
using System.Windows.Threading;
using Microsoft.Web.WebView2.Wpf;
using Octavia.Core;

namespace Octavia.Face;

/// The built-in page's fallback channel. It is used only when the page cannot reach
/// the WebSocket server; see PROTOCOL.md.
internal sealed class WebViewFaceTransport : IFaceTransport
{
    private readonly WebView2 _view;
    private readonly Dispatcher _dispatcher;

    /// The built-in page is a face like any other and needs an id to be addressable. It
    /// is minted once and never changes, because this transport is exactly one renderer.
    public FaceId Id { get; } = FaceId.New();

    public event Action<FaceMessage>? MessageReceived;

    public WebViewFaceTransport(WebView2 view)
    {
        _view = view;
        _dispatcher = view.Dispatcher;
        _view.CoreWebView2.WebMessageReceived += (_, e) =>
        {
            try
            {
                using var doc = JsonDocument.Parse(e.WebMessageAsJson);
                MessageReceived?.Invoke(new FaceMessage(Id, doc.RootElement.Clone()));
            }
            catch (Exception ex)
            {
                Log.Write($"bad message from face: {ex.Message}");
            }
        };
    }

    public FaceId? BuiltInFace => Id;

    public FaceStatus Status => new(Page: true, SocketBound: false, Port: 0, SocketFaces: 0);

    /// `to` is accepted and ignored: this transport is one face, so the only meaningful
    /// targets are "everyone" and "this one", and both mean the same thing here.
    public void Send(object message, FaceId? to = null) =>
        SendJson(JsonSerializer.Serialize(message, FaceHub.Json));

    public void SendJson(string json)
    {
        if (!_dispatcher.CheckAccess())
        {
            _dispatcher.BeginInvoke(() => SendJson(json));
            return;
        }

        try
        {
            _view.CoreWebView2?.PostWebMessageAsJson(json);
        }
        catch (Exception ex)
        {
            Log.Write($"send to face failed: {ex.Message}");
        }
    }
}
```
