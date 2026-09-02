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

namespace Octavia;

/// A window around her page, and nothing else.
///
/// Until Stage 15 this window *was* her: it built the WebView2, started the socket server,
/// constructed `OctaviaSession` and owned the being for as long as it was open. Now it
/// points a browser at a server and gets out of the way — which is what the Android client
/// has always done, and is why that client needed no changes to keep working.
///
/// Everything the page needs it learns from the origin it was loaded from. There are no
/// virtual hosts here any more: `octavia.face` and `octavia.avatar` were WebView2 features,
/// and the socket has served both roots over HTTP since v0.20.0.
public partial class MainWindow : Window
{
    private const int HotkeyId = 0xB0B;
    private const int WmHotkey = 0x0312;

    private readonly ClientConfig _client;
    private readonly string _hotkeyText;
    private readonly LocalServer.Outcome _server;
    private HwndSource? _hwnd;
    private bool _hotkeyRegistered;

    internal MainWindow(ClientConfig client, string hotkey, LocalServer.Outcome server)
    {
        _client = client;
        _hotkeyText = hotkey;
        _server = server;
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

        Face.CoreWebView2.Settings.AreDefaultContextMenusEnabled = false;
        Face.CoreWebView2.Settings.IsStatusBarEnabled = false;
        Face.CoreWebView2.Settings.IsSwipeNavigationEnabled = false;

        /* The host still answers every permission the page asks for rather than letting the
           runtime put its own prompt in front of a person.

           **What it can no longer do is second-guess the answer.** The camera setting is a
           property of a *room*, it lives in the session, and the session is in another
           process now — so a client that kept its own copy would be a second switch that
           disagreed with the first. The server is the gate: it decides whether `look` is
           ever sent, `hello.camera` decides whether the page offers a watch button, and
           `setCamera` is refused from any room but this one.

           So this narrows to the question the client can actually answer: is this her page,
           on the origin I was told to load? Anything else is denied. */
        Face.CoreWebView2.PermissionRequested += (_, request) =>
        {
            /* Camera and, since Stage 15 item 3, **microphone**.

               The page opens its own microphone now and streams it up, instead of asking the
               server to open the one on this machine — so a client that answered only for the
               camera would deny her page the very device the change exists to give it, and
               she would fall back to the server's microphone for ever without either of them
               saying why. */
            var mine = request.Uri.StartsWith(_client.Origin + "/", StringComparison.OrdinalIgnoreCase);

            var allowed = mine && request.PermissionKind is
                CoreWebView2PermissionKind.Camera or
                CoreWebView2PermissionKind.Microphone;

            request.State = allowed ? CoreWebView2PermissionState.Allow : CoreWebView2PermissionState.Deny;
            request.SavesInProfile = false;
            request.Handled = true;

            Log.Write($"permission {request.PermissionKind} from {request.Uri}: " +
                      $"{(allowed ? "allowed" : "denied")}");
        };

        /* Nothing is listening and we already know why, so there is no page to wait for.
           Spending the grace period proving something established before the window opened
           is thirty seconds of a person watching a blank rectangle. */
        if (_server is LocalServer.Outcome.Missing or LocalServer.Outcome.Failed)
        {
            Log.Error($"not attaching: {_client.Origin} has nothing behind it");
            ShowFallback(FaceFailureText());
            return;
        }

        Log.Write($"client attaching to {_client.Origin}");
        Face.CoreWebView2.Navigate(_client.PageUrl());
        WatchForFace();
    }

    /// Why her face is not on screen, in the words of the case that actually happened.
    ///
    /// **The old text named the rare cause first.** It offered a script syntax error before
    /// "nothing is serving her", which was the right order when this process *was* the
    /// server and a parse error was the only way the page could fail — and exactly the wrong
    /// order afterwards, when a shipped build's scripts have not changed since it was built
    /// and a missing server is the overwhelming case.
    ///
    /// It can name one cause now instead of guessing between two, because the client knows
    /// whether anything answered before it ever loaded a page.
    private string FaceFailureText() => _server switch
    {
        LocalServer.Outcome.Missing =>
            $"Nothing is serving her at {_client.Origin}.\n\nHer server is not running, and " +
            "there is no Octavia.Server.exe beside this client to start — so this looks like " +
            "a half-copied build. Start her server from its own shortcut, then open this " +
            "window again.",

        LocalServer.Outcome.Failed =>
            $"Her server was started, and never answered at {_client.Origin}.\n\nIt is " +
            "minimised in the taskbar and will say what stopped it; her log has the same " +
            "story with timestamps.",

        LocalServer.Outcome.Remote =>
            $"Nothing answered at {_client.Origin}.\n\nThat is another machine, so this " +
            "client cannot start her. Check her server is running there, and that the " +
            "address in client.json is the one you meant.",

        // Something *was* answering when the client looked, so the page reaching this window
        // and then failing to finish is a page fault again, and the old text is right again.
        _ => "Octavia's face did not start.\n\nHer server is answering, so her page reached " +
             "this window and then failed to finish — which usually means one of her script " +
             "files has a syntax error. The browser console will name it."
    };

    /// How long the page gets to load before the client stops believing in it.
    ///
    /// **It now means something different, and something narrower.** It used to mean "her
    /// renderer never reported in", because this process was also the host and knew whether
    /// `ready` had arrived. The client cannot know that any more — `ready` goes to a server
    /// it has no channel to. What it can still see is whether *its own page* got as far as
    /// running `bridge.js`, which is the fault this was built to catch: a JavaScript parse
    /// error is invisible to `dotnet build` and leaves a dead face and a green log.
    private static readonly TimeSpan FaceGrace = TimeSpan.FromSeconds(30);

    private void WatchForFace()
    {
        var timer = new System.Windows.Threading.DispatcherTimer { Interval = FaceGrace };

        timer.Tick += async (_, _) =>
        {
            timer.Stop();

            var alive = await Ask("typeof window.OctaviaFace === 'object'");
            if (alive == "true") return;

            Log.Error($"the page never finished loading within {FaceGrace.TotalSeconds:0}s");
            ShowFallback(FaceFailureText());
        };

        timer.Start();
    }

    /// Runs a snippet in the page and hands back its JSON result, or null if it could not
    /// be run at all. Every call into the renderer goes through here so that a page which
    /// is not there yet is a null rather than an exception on a timer thread.
    private async Task<string?> Ask(string script)
    {
        try
        {
            return Face.CoreWebView2 is null ? null : await Face.CoreWebView2.ExecuteScriptAsync(script);
        }
        catch (Exception ex)
        {
            Log.Write($"page script failed: {ex.Message}");
            return null;
        }
    }

    /// The tray and the hotkey reach her the only way this process can: through the face it
    /// is already showing. See `window.OctaviaFace` in `bridge.js`.
    private void Tell(string type) =>
        _ = Ask($"window.OctaviaFace && window.OctaviaFace.send({{ type: '{type}' }})");

    private void ShowFallback(string message)
    {
        Face.Visibility = Visibility.Collapsed;
        Fallback.Text = message;
        Fallback.Visibility = Visibility.Visible;
    }

    internal void ToggleListening() => Tell("listen");

    /// A crash the user never sees is a crash that gets reported as "it just sits there".
    /// This one is the *client's* — hers arrive as notices from the server like any other.
    internal void ReportCrash(Exception ex)
    {
        var text = System.Text.Json.JsonSerializer.Serialize(
            $"Something went wrong in her window: {ex.Message}. It is in the log.");

        _ = Ask($"window.OctaviaFace && window.OctaviaFace.notify({text})");
    }

    /// Reachable from the tray as well as the face, because the times you most need a
    /// diagnostics bundle are the times the face is the thing that broke. The server writes
    /// it and says where in a notice — there is no dialog to show, and could not be: the
    /// process that writes the file may not be on this machine.
    internal void SaveDiagnostics() => Tell("saveDiagnostics");

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

        if (!Hotkey.TryParse(_hotkeyText, out var hotkey))
        {
            Log.Write($"hotkey '{_hotkeyText}' is not a combination I understand");
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
    }
}
```
