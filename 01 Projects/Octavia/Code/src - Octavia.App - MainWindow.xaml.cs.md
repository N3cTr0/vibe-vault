---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\MainWindow.xaml.cs
---

# src\Octavia.App\MainWindow.xaml.cs

```csharp
using System.ComponentModel;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;
using Microsoft.Web.WebView2.Core;
using Octavia.Core;
using Octavia.Face;

namespace Octavia;

public partial class MainWindow : Window
{
    private const string FaceHost = "octavia.face";
    private const string AvatarHost = "octavia.avatar";
    private const int HotkeyId = 0xB0B;
    private const int WmHotkey = 0x0312;

    private readonly OctaviaConfig _config;
    private OctaviaSession? _session;
    private WebSocketFaceServer? _sockets;
    private FaceHub? _hub;
    private HwndSource? _hwnd;
    private bool _hotkeyRegistered;

    internal MainWindow(OctaviaConfig config)
    {
        _config = config;
        InitializeComponent();
        Loaded += OnLoaded;
        Closing += OnClosing;
    }

    /// True once the window has been asked to close for real rather than to the tray.
    internal bool ShuttingDown { get; set; }

    private async void OnLoaded(object sender, RoutedEventArgs e)
    {
        RegisterHotkey();

        try
        {
            var environment = await CoreWebView2Environment.CreateAsync(
                userDataFolder: Paths.BrowserData);

            await Face.EnsureCoreWebView2Async(environment);
        }
        catch (Exception ex)
        {
            Log.Write($"WebView2 unavailable: {ex}");
            ShowFallback(
                "Octavia needs the Microsoft Edge WebView2 runtime, which does not appear to be " +
                "installed.\n\nInstall the Evergreen runtime from Microsoft, then start her again.");
            return;
        }

        // A virtual https origin, not file://, so the page is a secure context. The camera
        // and microphone permissions that come later depend on that.
        Face.CoreWebView2.SetVirtualHostNameToFolderMapping(
            FaceHost, Paths.FaceRoot, CoreWebView2HostResourceAccessKind.Allow);

        // Characters live in her data folder, not the install, so they need an origin of
        // their own. Read-only, and only that one folder is reachable.
        Face.CoreWebView2.SetVirtualHostNameToFolderMapping(
            AvatarHost, Paths.AvatarDir, CoreWebView2HostResourceAccessKind.Allow);

        Face.CoreWebView2.Settings.AreDefaultContextMenusEnabled = false;
        Face.CoreWebView2.Settings.IsStatusBarEnabled = false;
        Face.CoreWebView2.Settings.IsSwipeNavigationEnabled = false;

        // The host answers every permission the page asks for, rather than letting the
        // renderer negotiate with the runtime behind its back. Left unhandled, WebView2
        // would put its own prompt in front of a person for anything a page requested —
        // which makes "Camera": false a suggestion rather than a boundary.
        Face.CoreWebView2.PermissionRequested += (_, request) =>
        {
            // Only her own page may ask at all. Nothing else is navigable today, but a
            // permission handler that trusts its caller is one bug away from mattering.
            var mine = request.Uri.StartsWith($"https://{FaceHost}/", StringComparison.OrdinalIgnoreCase);

            var allowed = mine
                       && request.PermissionKind == CoreWebView2PermissionKind.Camera
                       && _config.Camera;

            request.State = allowed ? CoreWebView2PermissionState.Allow : CoreWebView2PermissionState.Deny;

            // Not saved in the profile: the answer is whatever config says *now*, so
            // turning the camera off takes effect without clearing browser state.
            request.SavesInProfile = false;
            request.Handled = true;

            Log.Write($"permission {request.PermissionKind} from {request.Uri}: " +
                      $"{(allowed ? "allowed" : "denied")}");
        };

        _sockets = new WebSocketFaceServer();
        var socketUp = _sockets.Start(_config.FacePort, _config.RemoteAccess);

        _hub = new FaceHub(new WebViewFaceTransport(Face), socketUp ? _sockets : null);
        _session = new OctaviaSession(_config, _hub);

        // The page is told where to connect and with what token. Without these it
        // silently falls back to postMessage, which still works but cannot be shared.
        var address = socketUp
            ? $"https://{FaceHost}/index.html?port={_sockets.Port}&token={_sockets.Token}"
            : $"https://{FaceHost}/index.html";

        Face.CoreWebView2.Navigate(address);
    }

    private void ShowFallback(string message)
    {
        Face.Visibility = Visibility.Collapsed;
        Fallback.Text = message;
        Fallback.Visibility = Visibility.Visible;
    }

    internal void ToggleListening() => _session?.ToggleListening();

    /// Reachable from the tray as well as the face, because the times you most need a
    /// diagnostics bundle are the times the face is the thing that broke.
    internal void SaveDiagnostics() => _session?.SaveDiagnosticsAsync().Forget("saving diagnostics");

    /// A crash the user never sees is a crash that gets reported as "it just sits there".
    internal void ReportCrash(Exception ex) => _hub?.Send(new
    {
        type = "notice",
        text = $"Something went wrong: {ex.Message}. It is in the log — use Diagnostics to package it."
    });

    internal void Surface()
    {
        Show();
        if (WindowState == WindowState.Minimized) WindowState = WindowState.Normal;
        Activate();
    }

    private void RegisterHotkey()
    {
        _hwnd = (HwndSource?)PresentationSource.FromVisual(this);
        if (_hwnd is null) return;

        _hwnd.AddHook(OnWindowMessage);

        if (!Hotkey.TryParse(_config.Hotkey, out var hotkey))
        {
            Log.Write($"hotkey '{_config.Hotkey}' is not a combination I understand");
            return;
        }

        if (Native.RegisterHotKey(_hwnd.Handle, HotkeyId, hotkey.Modifiers, hotkey.VirtualKey))
        {
            _hotkeyRegistered = true;
            Log.Write($"hotkey {hotkey.Display} registered");
            return;
        }

        var error = Marshal.GetLastWin32Error();
        Log.Write(error == 1409
            ? $"hotkey {hotkey.Display} is already owned by another application; change Hotkey in config.json"
            : $"hotkey {hotkey.Display} failed to register (win32 {error})");
    }

    private nint OnWindowMessage(nint hwnd, int msg, nint wParam, nint lParam, ref bool handled)
    {
        if (msg != WmHotkey || wParam != HotkeyId) return nint.Zero;
        Surface();
        ToggleListening();
        handled = true;
        return nint.Zero;
    }

    private void OnClosing(object? sender, CancelEventArgs e)
    {
        if (!ShuttingDown)
        {
            // Closing the window puts her in the tray; quitting is done from there.
            e.Cancel = true;
            Hide();
            return;
        }

        if (_hwnd is not null)
        {
            if (_hotkeyRegistered) Native.UnregisterHotKey(_hwnd.Handle, HotkeyId);
            _hwnd.RemoveHook(OnWindowMessage);
        }

        _session?.Dispose();
        _hub?.Dispose();
        _sockets?.Dispose();
    }
}
```
