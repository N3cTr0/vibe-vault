---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\SyntaxChecks.cs
---

# tools\EarsTest\SyntaxChecks.cs

```csharp
// The gap this closes: a JavaScript syntax error in wwwroot is invisible to `dotnet build`
// and to every other check in this harness, so the build goes green and the face is simply
// dead. It happened in v0.18.0 — a `sed` took the wrong four lines, left an orphan `});` in
// bridge.js, and the only symptom was that a button stopped working. See ROADMAP.md stage 10a.
//
// This does not hand-roll a parser. It loads the real page in the WebView2 the app already
// depends on and lets Chromium do the parsing, which is both dependency-free and the same
// engine that will run it for real. It is therefore a slightly larger claim than "the syntax
// is valid": it is "the face comes up".
using System.Runtime.InteropServices;
using System.Text.Json;
using Microsoft.Web.WebView2.Core;
using Microsoft.Web.WebView2.WinForms;
using Octavia.Core;

internal static class SyntaxChecks
{
    private const string FaceHost = "octavia.face";

    /// Generous: this has to build a WebGL scene on whatever GPU is present.
    private static readonly TimeSpan Budget = TimeSpan.FromSeconds(45);

    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        var root = Paths.FaceRoot;
        if (!Directory.Exists(root))
        {
            Check("wwwroot is where it should be", false, root);
            return failures;
        }

        // Our own files, not the vendored library. A broken three.js would be caught by the
        // page load below anyway; listing ours is what makes the count meaningful.
        var ours = Directory.GetFiles(root, "*.js", SearchOption.TopDirectoryOnly)
                            .Select(Path.GetFileName)
                            .OrderBy(n => n)
                            .ToList();

        Check("there are scripts to check", ours.Count > 0, "none found");
        if (ours.Count == 0) return failures;

        Result result;
        try
        {
            result = LoadPage(root);
        }
        catch (Exception ex)
        {
            // A machine with no WebView2 runtime cannot run this check, and that is not a
            // failure of the face. Say so plainly rather than reporting a fault that is not there.
            Console.WriteLine($"  ..   skipped: WebView2 would not start ({ex.GetType().Name}: {ex.Message})");
            return failures;
        }

        Check($"the page loaded ({ours.Count} scripts: {string.Join(", ", ours)})",
            result.Navigated, result.Detail);

        // The whole point. A parse error reaches the console as a SyntaxError and nothing else does.
        Check("no script failed to parse", result.SyntaxErrors.Count == 0,
            string.Join(" | ", result.SyntaxErrors));

        // face.js sets window.Face as it evaluates, so this is dead if any module in the
        // chain failed — including one that parsed and then threw.
        Check("the renderer published window.Face", result.FacePresent, "window.Face is not an object");

        // What the host waits 30s for at startup. If it is false here it would be false there.
        Check("the bridge would have sent 'ready'", result.BridgeReady, result.Detail);

        if (result.OtherErrors.Count > 0)
            Console.WriteLine($"  ..   {result.OtherErrors.Count} non-syntax console error(s): " +
                              string.Join(" | ", result.OtherErrors.Take(3)));

        return failures;
    }

    private sealed record Result(
        bool Navigated,
        bool FacePresent,
        bool BridgeReady,
        List<string> SyntaxErrors,
        List<string> OtherErrors,
        string Detail);

    /// WebView2 needs a real window and a message pump, so this runs one on its own STA
    /// thread and blocks until it has an answer. The window is positioned off-screen rather
    /// than hidden: a hidden WebView2 does not run WebGL or requestAnimationFrame, which
    /// would make the scene fail to build for reasons that have nothing to do with the code.
    private static Result LoadPage(string root)
    {
        Result? outcome = null;
        Exception? failure = null;

        var thread = new Thread(() =>
        {
            try { outcome = Pump(root); }
            catch (Exception ex) { failure = ex; }
        });

        thread.SetApartmentState(ApartmentState.STA);
        thread.IsBackground = true;
        thread.Start();

        if (!thread.Join(Budget + TimeSpan.FromSeconds(20)))
            throw new TimeoutException("the WebView2 thread never finished");

        if (failure is not null) throw failure;
        return outcome ?? throw new InvalidOperationException("no result");
    }

    private static Result Pump(string root)
    {
        var syntax = new List<string>();
        var other = new List<string>();
        var navigated = false;
        var detail = "";
        var facePresent = false;
        var bridgeReady = false;

        using var form = new System.Windows.Forms.Form
        {
            // Off-screen, not hidden. See LoadPage.
            StartPosition = System.Windows.Forms.FormStartPosition.Manual,
            Location = new System.Drawing.Point(-32000, -32000),
            ShowInTaskbar = false,
            Width = 900,
            Height = 700
        };

        using var view = new WebView2 { Dock = System.Windows.Forms.DockStyle.Fill };
        form.Controls.Add(view);

        using var host = new PageHost();
        var done = new ManualResetEventSlim(false);

        form.Shown += async (_, _) =>
        {
            try
            {
                var environment = await CoreWebView2Environment.CreateAsync(
                    userDataFolder: Path.Combine(Path.GetTempPath(), "octavia-syntaxcheck"));

                await view.EnsureCoreWebView2Async(environment);

                await view.CoreWebView2.AddScriptToExecuteOnDocumentCreatedAsync(
                    """
                    window.__octaviaErrors = [];
                    window.addEventListener('error', e => {
                      const err = e.error;
                      window.__octaviaErrors.push({
                        name: (err && err.name) || 'Error',
                        message: String((err && err.message) || e.message || ''),
                        where: String(e.filename || '') + ':' + (e.lineno || 0)
                      });
                    });
                    """);

                view.CoreWebView2.NavigationCompleted += async (_, e) =>
                {
                    navigated = e.IsSuccess;
                    if (!e.IsSuccess) detail = e.WebErrorStatus.ToString();

                    // The page loads its modules after navigation completes, and bridge.js
                    // sends `ready` only once its transport settles. Give both a moment.
                    await Task.Delay(3500);

                    bridgeReady = host.Heard().Contains("\"type\":\"ready\"");

                    try
                    {
                        var errors = await view.CoreWebView2.ExecuteScriptAsync(
                            "JSON.stringify(window.__octaviaErrors || [])");
                        var face = await view.CoreWebView2.ExecuteScriptAsync(
                            "JSON.stringify(typeof window.Face === 'object' && window.Face !== null)");

                        facePresent = face.Contains("true");

                        // ExecuteScriptAsync returns the value JSON-encoded, so the string
                        // comes back wrapped and escaped; unwrap once before parsing it.
                        var raw = JsonSerializer.Deserialize<string>(errors) ?? "[]";
                        using var doc = JsonDocument.Parse(raw);

                        foreach (var item in doc.RootElement.EnumerateArray())
                        {
                            var name = item.GetProperty("name").GetString() ?? "Error";
                            var line = $"{name}: {item.GetProperty("message").GetString()} " +
                                       $"({item.GetProperty("where").GetString()})";

                            if (name == "SyntaxError") syntax.Add(line);
                            else other.Add(line);
                        }
                    }
                    catch (Exception ex)
                    {
                        detail = $"could not read the page back: {ex.Message}";
                    }

                    done.Set();
                };

                /* Served by a real socket over loopback, which is what the desktop client
                   now does — and loopback HTTP is a secure context, so the page gets the
                   same CSP and the same `getUserMedia` availability the virtual `https`
                   origin used to give it.

                   It has to be a real server rather than a virtual host, because `ready` is
                   the thing being checked and `ready` is sent from the socket's `open`
                   handler. A page with nothing to connect to correctly says nothing. */
                view.CoreWebView2.Navigate(host.Url);
            }
            catch (Exception ex)
            {
                detail = ex.Message;
                done.Set();
            }
        };

        // A message-only pump: show the form off-screen, run until the load reports back,
        // then close it. Application.Run would need a matching Exit from the callback.
        form.Show();

        var deadline = DateTime.UtcNow + Budget;
        while (!done.IsSet && DateTime.UtcNow < deadline)
        {
            System.Windows.Forms.Application.DoEvents();
            Thread.Sleep(25);
        }

        if (!done.IsSet && detail.Length == 0) detail = $"nothing reported back within {Budget.TotalSeconds:0}s";

        form.Close();

        return new Result(navigated, facePresent, bridgeReady, syntax, other, detail);
    }
}
```
