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

    public event Action<JsonElement>? MessageReceived;

    public WebViewFaceTransport(WebView2 view)
    {
        _view = view;
        _dispatcher = view.Dispatcher;
        _view.CoreWebView2.WebMessageReceived += (_, e) =>
        {
            try
            {
                using var doc = JsonDocument.Parse(e.WebMessageAsJson);
                MessageReceived?.Invoke(doc.RootElement.Clone());
            }
            catch (Exception ex)
            {
                Log.Write($"bad message from face: {ex.Message}");
            }
        };
    }

    public FaceStatus Status => new(Page: true, SocketBound: false, Port: 0, SocketFaces: 0);

    public void Send(object message) => SendJson(JsonSerializer.Serialize(message, FaceHub.Json));

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
