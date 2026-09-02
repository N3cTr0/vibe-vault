---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\EmbedderChecks.cs
---

# tools\EarsTest\EmbedderChecks.cs

```csharp
// Stage 14 item 10 — a renderer borrowing the senses of whatever it is embedded in.
//
// Item 9 hid two controls on a face outside the host room, both for good reasons: the
// microphone sends `listen`, which drives the *host machine's* microphone, and the watch
// button needs `getUserMedia`, which does not exist outside a secure context. Both were
// right and both left a handset — which has a microphone and a camera — offered neither.
//
// Neither is fixable on the wire. The floor is a `FaceId`, so the panel cannot press while
// the native connection streams; and watching is renderer-local by design and should stay
// there. So the page looks for `window.OctaviaEmbedder` and borrows.
//
// **Every acceptance criterion for this lives in the page**, so this drives the real page in
// the real engine rather than asserting anything about C#. The embedder is injected with
// `AddScriptToExecuteOnDocumentCreatedAsync`, which is where a WebView host would put it.
using System.Runtime.InteropServices;
using System.Text.Json;
using Microsoft.Web.WebView2.Core;
using Microsoft.Web.WebView2.WinForms;
using Octavia.Core;

internal static class EmbedderChecks
{
    private const string FaceHost = "octavia.face";
    private static readonly TimeSpan Budget = TimeSpan.FromSeconds(60);

    internal readonly record struct Finding(string Name, bool Ok, string Detail);

    public static int Run()
    {
        var failures = 0;

        void Report(Finding f)
        {
            Console.WriteLine($"  {(f.Ok ? "ok  " : "FAIL")} {f.Name}{(f.Ok ? "" : $" — {f.Detail}")}");
            if (!f.Ok) failures++;
        }

        var root = Paths.FaceRoot;
        if (!Directory.Exists(root))
        {
            Report(new Finding("wwwroot is where it should be", false, root));
            return failures;
        }

        List<Finding> findings;
        try
        {
            findings = Drive(root);
        }
        catch (Exception ex)
        {
            // A machine with no WebView2 runtime cannot run this, and that is not a failure
            // of the face. Say so plainly rather than reporting a fault that is not there.
            Console.WriteLine($"  ..   skipped: WebView2 would not start ({ex.GetType().Name}: {ex.Message})");
            return failures;
        }

        foreach (var f in findings) Report(f);
        return failures;
    }

    private static List<Finding> Drive(string root)
    {
        List<Finding>? outcome = null;
        Exception? failure = null;

        var thread = new Thread(() =>
        {
            try { outcome = Pump(root); }
            catch (Exception ex) { failure = ex; }
        });

        thread.SetApartmentState(ApartmentState.STA);
        thread.IsBackground = true;
        thread.Start();

        if (!thread.Join(Budget + TimeSpan.FromSeconds(30)))
            throw new TimeoutException("the WebView2 thread never finished");

        if (failure is not null) throw failure;
        return outcome ?? throw new InvalidOperationException("no result");
    }

    /// A handset: an embedder lending both senses, on a face outside the host room.
    private const string Embedder = """
        window.__calls = [];
        window.OctaviaEmbedder = {
          senses: ['mic', 'camera'],
          talking(on) { window.__calls.push('talking:' + on); },
          watch(on) { window.__calls.push('watch:' + on); }
        };
        """;

    private static List<Finding> Pump(string root)
    {
        var findings = new List<Finding>();

        void Check(string name, bool ok, string detail = "") =>
            findings.Add(new Finding(name, ok, detail));

        using var form = new System.Windows.Forms.Form
        {
            // Off-screen rather than hidden: a hidden WebView2 does not run WebGL or
            // requestAnimationFrame, so the scene would fail to build for reasons that have
            // nothing to do with this code. The same rule as SyntaxChecks.
            StartPosition = System.Windows.Forms.FormStartPosition.Manual,
            Location = new System.Drawing.Point(-32000, -32000),
            ShowInTaskbar = false,
            Width = 900,
            Height = 700
        };

        using var view = new WebView2 { Dock = System.Windows.Forms.DockStyle.Fill };
        form.Controls.Add(view);

        /* A real socket, because after Stage 15 that is the only way into this page.

           These checks used to be the host: `PostWebMessageAsJson` a `hello` in, read what
           came back off `WebMessageReceived`. That channel is gone, so the harness speaks
           the protocol instead — which is what a phone and the desktop client both do, and
           therefore a fairer test than the one it replaces. */
        using var host = new PageHost();

        var done = new ManualResetEventSlim(false);

        form.Shown += async (_, _) =>
        {
            try
            {
                var environment = await CoreWebView2Environment.CreateAsync(
                    userDataFolder: Path.Combine(Path.GetTempPath(), "octavia-embeddercheck"));

                await view.EnsureCoreWebView2Async(environment);

                /* **Two origins, and the difference is the point.**

                   The desktop client loads her page from the server over loopback, and
                   loopback HTTP *is* a secure context — so `getUserMedia` exists there. A
                   phone loads the same page from the same server over the LAN, where it is
                   not, and `getUserMedia` is simply absent.

                   Loopback cannot be made insecure, so the LAN case keeps a virtual host to
                   get a non-loopback hostname and reaches the real socket with `?port=`. The
                   check below asserts which context it actually got, so a run that passes
                   for the wrong reason says so. */
                view.CoreWebView2.SetVirtualHostNameToFolderMapping(
                    FaceHost, root, CoreWebView2HostResourceAccessKind.Allow);

                /// `ExecuteScriptAsync` hands back the completion value JSON-encoded, so a
                /// string arrives wrapped and escaped and has to be unwrapped once. Anything
                /// that is not a string — `window.__calls = []` evaluates to `[]` — is
                /// returned as it came, rather than throwing halfway through the run and
                /// reporting it as the checks failing to start.
                async Task<string> Eval(string js)
                {
                    var raw = await view.CoreWebView2.ExecuteScriptAsync(js);
                    try { return JsonSerializer.Deserialize<string>(raw) ?? ""; }
                    catch (JsonException) { return raw ?? ""; }
                }

                /* Loads the page fresh with — or without — an embedder in front of it.

                   The previous script is **removed** rather than merely superseded.
                   `AddScriptToExecuteOnDocumentCreated` accumulates: without the removal the
                   handset's embedder would still be injected into the "plain browser" run,
                   and the two checks that prove a browser tab is left alone would pass while
                   testing the wrong page entirely. */
                string? injected = null;

                /// The LAN: a non-loopback hostname, so no secure context, reaching the real
                /// socket with `?port=`.
                string Lan() => $"http://{FaceHost}/index.html{host.Query}";

                async Task Load(string url, string? inject)
                {
                    if (injected is not null)
                    {
                        view.CoreWebView2.RemoveScriptToExecuteOnDocumentCreated(injected);
                        injected = null;
                    }

                    view.CoreWebView2.Navigate("about:blank");
                    await Task.Delay(400);

                    injected = await view.CoreWebView2.AddScriptToExecuteOnDocumentCreatedAsync(
                        """
                        window.__octaviaErrors = [];
                        window.addEventListener('error', e => window.__octaviaErrors.push(
                          String((e.error && e.error.message) || e.message || '')));
                        """ + (inject ?? ""));

                    host.Clear();
                    view.CoreWebView2.Navigate(url);

                    // The face has to reach the socket before there is anything to say to
                    // it, and `ready` is the first thing it says once it has.
                    host.Wait("ready", TimeSpan.FromSeconds(20));
                    await Task.Delay(1200);
                }

                // A `hello` complete enough that the page's handler runs the whole way
                // through. Anything it does not name is guarded on the page.
                string Hello(string controls, bool camera) => JsonSerializer.Serialize(new
                {
                    type = "hello",
                    protocol = 1,
                    room = controls == "host" ? "host" : "phone",
                    controls,
                    hasKey = true,
                    model = "stub",
                    profile = "test",
                    ears = "not started",
                    voice = "Amy",
                    voices = Array.Empty<object>(),
                    voiceEngine = "neural",
                    listening = false,
                    avatarFile = "",
                    avatars = Array.Empty<string>(),
                    roomHour = -1,
                    music = false,
                    musicAvailable = false,
                    camera,
                    cameraDevice = "",
                    stats = true,
                    whisperCompute = "auto",
                    toolServers = Array.Empty<object>(),
                    audioAvailable = true,
                    audioRate = 22050,
                    audioBits = 16,
                    audioChannels = 1,
                    micAccepted = true,
                    dev = false,
                    state = "idle",
                    emotion = "neutral",
                    emotionWeight = 0.0
                });

                async Task Say(string controls, bool camera)
                {
                    host.Send(Hello(controls, camera), "hello");
                    await Task.Delay(600);
                }

                async Task<string> Calls() => await Eval("JSON.stringify(window.__calls)");

                async Task Press(string action) =>
                    await Eval($$"""
                        (() => {
                          const b = document.getElementById('talk');
                          b.dispatchEvent(new PointerEvent('{{action}}',
                            { bubbles: true, cancelable: true, pointerId: 1 }));
                          return 'ok';
                        })()
                        """);

                // ---- a handset: an embedder, outside the host room ------------------
                await Load(Lan(), Embedder);
                await Say("room", camera: true);

                Check("the LAN origin really is insecure",
                    await Eval("String(window.isSecureContext)") == "false",
                    "the check would prove nothing if it were secure");

                Check("a borrowed microphone brings the button back",
                    await Eval("String(!document.getElementById('talk').hidden)") == "true",
                    "the microphone button stayed hidden on a face that has one");

                Check("...and says it is held rather than toggled",
                    (await Eval("document.getElementById('talk').getAttribute('aria-label')")) == "Hold to talk",
                    await Eval("document.getElementById('talk').getAttribute('aria-label')"));

                Check("a borrowed camera brings the watch button back",
                    await Eval("String(!document.getElementById('watch').hidden)") == "true",
                    "hidden, on a device holding a camera");

                /* **The hold does not begin the instant the finger lands**, since item 6 gave
                   the same button a tap as well: starting immediately and cancelling on a
                   quick release would take and drop the floor on every tap. So every press
                   check has to outwait `HOLD_AFTER_MS`, and the delay being *wrong* is worth
                   catching too — a hold that never engages is a microphone button that does
                   nothing. */
                await Press("pointerdown");
                await Task.Delay(600);

                Check("pressing takes the floor through the embedder",
                    (await Calls()).Contains("talking:true"),
                    await Calls());

                Check("...and the button shows it is held",
                    await Eval("document.getElementById('talk').getAttribute('aria-pressed')") == "true",
                    "aria-pressed did not follow the finger");

                await Press("pointerup");
                Check("releasing releases",
                    (await Calls()).Contains("talking:false"),
                    await Calls());

                /* The failure this must not have. A held button that never releases holds
                   her ears until the host's sixty-second floor timeout, and every way a
                   press can end has to lead to the same place. */
                await Eval("window.__calls = []");
                await Press("pointerdown");
                await Task.Delay(600);
                await Press("pointerleave");
                Check("dragging off the button releases too",
                    (await Calls()).Contains("talking:false"),
                    await Calls());

                await Eval("window.__calls = []");
                await Press("pointerdown");
                await Task.Delay(600);
                await Press("pointercancel");
                Check("the system taking the gesture releases too",
                    (await Calls()).Contains("talking:false"),
                    await Calls());

                /* A tap is not a press, and must not become one. Releasing before the hold
                   engages is how somebody says "leave this listening"; if it took the floor
                   on the way past, every tap would be a quarter-second of her attention and
                   a line in her log. This embedder has no `listening`, so the tap has
                   nowhere to go — which is the point: it still must not talk. */
                await Eval("window.__calls = []");
                await Press("pointerdown");
                await Press("pointerup");
                await Task.Delay(600);

                Check("a tap is not a press",
                    !(await Calls()).Contains("talking"),
                    await Calls());

                // The whole reason the button is hidden without an embedder: `listen` drives
                // the host machine's microphone, and a room face must never send it.
                var sent = host.Heard();
                Check("none of that sent `listen` to the host",
                    !sent.Contains("\"listen\""), sent);

                // Borrowed senses are this renderer's own business. Claiming a camera to the
                // host would have `look` sent to a panel that cannot take a still — and on a
                // handset it would take that frame away from the native client, which can.
                Check("a borrowed camera is not claimed to the host",
                    sent.Contains("\"ready\"") && !sent.Contains("\"camera\""),
                    sent.Length > 0 ? sent : "no ready was seen at all");

                // ---- watching, through the embedder ---------------------------------
                await Eval("window.__calls = []");
                await Eval("document.getElementById('watch').click(); 'ok'");
                await Task.Delay(600);

                Check("watching borrows the camera too",
                    (await Calls()).Contains("watch:true"),
                    await Calls());

                Check("...and the page keeps the privacy marker",
                    await Eval("String(!document.getElementById('watching').hidden)") == "true",
                    "the marker never came up, so nothing says the camera is live");

                await Eval("document.getElementById('watch').click(); 'ok'");
                await Task.Delay(400);

                Check("pressing again stops it",
                    (await Calls()).Contains("watch:false"),
                    await Calls());

                Check("...and the marker goes down with it",
                    await Eval("String(document.getElementById('watching').hidden)") == "true",
                    "the marker stayed up over a camera that had stopped");

                // ---- a plain browser on the LAN: no embedder ------------------------
                await Load(Lan(), null);
                await Say("room", camera: true);

                Check("with no embedder the microphone button stays hidden",
                    await Eval("String(document.getElementById('talk').hidden)") == "true",
                    "a browser tab was offered a button that sends `listen`");

                Check("with no embedder the watch button stays hidden",
                    await Eval("String(document.getElementById('watch').hidden)") == "true",
                    "a control that could only throw");

                Check("and nothing threw",
                    await Eval("String((window.__octaviaErrors || []).length)") == "0",
                    await Eval("JSON.stringify(window.__octaviaErrors || [])"));

                // ---- the desktop: host room, unchanged ------------------------------
                await Load(host.Url, null);
                await Say("host", camera: false);

                Check("the host room still has its microphone button",
                    await Eval("String(!document.getElementById('talk').hidden)") == "true",
                    "item 10 took the desktop's button away");

                Check("...and it still toggles rather than holds",
                    (await Eval("document.getElementById('talk').getAttribute('aria-label')")) == "Toggle listening",
                    await Eval("document.getElementById('talk').getAttribute('aria-label')"));

                host.Clear();
                await Eval("document.getElementById('talk').click(); 'ok'");

                // Longer than the others, because the click now tries to *open a microphone*
                // before it decides anything, and being refused one is not instant.
                await Task.Delay(4000);

                /* **This asserts the fallback, and it did not used to.** Before Stage 15
                   item 3 the desktop's button always sent `listen` — open the server's
                   microphone. It now prefers its own, and only asks for hers when that
                   fails.

                   There is no capture device in this harness, so opening one fails and the
                   fallback is exactly what runs. That makes this the check for the case that
                   matters most: a desk whose microphone is denied, unplugged or already
                   taken must not go silent. The path where the page *succeeds* needs a real
                   microphone and is exercised by hand. */
                sent = host.Heard();
                Check("...and falls back to `listen` when it has no microphone of its own",
                    sent.Contains("\"listen\""), sent);
            }
            catch (Exception ex)
            {
                Check("the checks ran at all", false, $"{ex.GetType().Name}: {ex.Message}");
            }
            finally
            {
                done.Set();
            }
        };

        form.Show();

        var deadline = DateTime.UtcNow + Budget;
        while (!done.IsSet && DateTime.UtcNow < deadline)
        {
            System.Windows.Forms.Application.DoEvents();
            Thread.Sleep(25);
        }

        if (!done.IsSet)
            findings.Add(new Finding("the checks finished", false,
                $"nothing reported back within {Budget.TotalSeconds:0}s"));

        form.Close();
        return findings;
    }
}
```
