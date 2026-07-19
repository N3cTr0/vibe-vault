---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\App.xaml.cs
---

# PartnerTool\App.xaml.cs

```csharp
using System.Diagnostics;
using System.Globalization;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Interop;
using System.Windows.Markup;
using System.Windows.Media;
using System.Windows.Threading;

namespace PartnerTool;

/// <summary>
/// Interaction logic for App.xaml
/// </summary>
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        // Session 0 = hidden service sessions (NinjaOne Background Mode / "Backstage"). WPF cannot
        // rasterize there AT ALL — proven empirically 2026-07-14 with a five-mode probe: hardware,
        // software (per-window and process-wide), layered windows, and even offscreen
        // RenderTargetBitmap→GDI-blit all produce zero pixels (the render thread never comes up),
        // while pure GDI painting displays fine. So don't sit there as a black rectangle: tell the
        // tech with a native GDI message box (the one dialog type that DOES render there) and exit.
        if (BlockHiddenServiceSession(e.Args)) { Shutdown(); return; }

        // The UI and docs are American English, but the tech's machine can be set to any locale.
        // Without pinning the culture, numbers render with the OS decimal separator (e.g. "10,0 / 10",
        // "5,5 GB") and dates as d/M/yy — inconsistent and confusing on a US-English tool. Force en-US
        // for all formatting on the UI thread, background threads, and WPF binding StringFormat.
        ForceEnUsCulture();

        // Elsewhere-but-not-console (RDP, odd remote stacks): WPF works, but hardware presentation
        // can glitch — fall back to the CPU rasterizer. Free on a normal desktop (sessions match).
        // `-softwarerender` forces it anywhere, for broken GPU drivers.
        ApplyRenderModeFallback(e.Args);

        // WPF emits "used controls" telemetry during shutdown
        // (MS.Internal.Telemetry…ControlsTraceLogger.LogUsedControlsDetails). In a single-file
        // publish the runtime can't resolve System.Diagnostics.Tracing by name and throws
        // FileNotFoundException as the app closes — crashing the process on every exit (the dev
        // build is unaffected because the DLL is present on disk). Satisfy just that one request
        // from the assembly that actually provides EventSource so shutdown completes cleanly.
        AppDomain.CurrentDomain.AssemblyResolve += OnAssemblyResolve;

        // Lock down C:\PCI before anything writes to or trusts it. The elevated app logs here and
        // downloads/executes vendor tools from C:\PCI\Tools; default C:\ ACLs would otherwise let a
        // standard user pre-own these folders and plant files we'd run as admin.
        SecureDirectory.EnsureHardened(@"C:\PCI");
        SecureDirectory.EnsureHardened(@"C:\PCI\Logs");

        // This tool runs across many different partner machines. A single WMI,
        // registry or process hiccup inside an async handler shouldn't take the
        // whole app down — catch it, tell the tech, and keep running.
        DispatcherUnhandledException += OnUnhandledException;

        // Defence in depth: a stray exception on a background thread or an unobserved
        // task would otherwise be able to take the process down. Log these so we can
        // diagnose them, and mark task exceptions observed so they don't escalate.
        AppDomain.CurrentDomain.UnhandledException += (_, ex) => LogCrash("AppDomain", ex.ExceptionObject as Exception);
        TaskScheduler.UnobservedTaskException += (_, ex) => { LogCrash("Task", ex.Exception); ex.SetObserved(); };

        // Load (or create) settings.json before the window builds its pages.
        SettingsStore.Load();

        // Housekeeping: prune old logs from the hardened C:\PCI\Logs (only the elevated app can) —
        // in the background, so a slow disk never delays the splash.
        _ = Task.Run(LogRetention.Prune);

        base.OnStartup(e);
    }

    /// <summary>
    /// Pin every formatting path to en-US so numbers and dates read the same on any machine's locale.
    /// Covers the UI thread, the default culture new threads/Tasks inherit, and WPF's binding
    /// StringFormat (which reads FrameworkElement.Language, not CurrentCulture).
    /// </summary>
    private static void ForceEnUsCulture()
    {
        var enUs = new CultureInfo("en-US");
        // House standard is MM/DD/YYYY (see Dates.cs). en-US defaults to M/d/yyyy, so pin the
        // short-date pattern — this backstops any default DateTime.ToString() / {0:d} binding.
        enUs.DateTimeFormat.ShortDatePattern = "MM/dd/yyyy";
        CultureInfo.DefaultThreadCurrentCulture   = enUs;
        CultureInfo.DefaultThreadCurrentUICulture = enUs;
        Thread.CurrentThread.CurrentCulture   = enUs;
        Thread.CurrentThread.CurrentUICulture = enUs;
        FrameworkElement.LanguageProperty.OverrideMetadata(
            typeof(FrameworkElement),
            new FrameworkPropertyMetadata(XmlLanguage.GetLanguage(enUs.IetfLanguageTag)));
    }

    /// <summary>True when the Backstage/hidden-session fallback put WPF into software rendering.</summary>
    public static bool SoftwareRendering { get; private set; }

    [DllImport("kernel32.dll")]
    private static extern uint WTSGetActiveConsoleSessionId();

    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    private static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);

    /// <summary>
    /// True when we're in session 0 — a hidden service session (e.g. NinjaOne Background Mode)
    /// where WPF provably cannot draw. Shows a native GDI notice (which does render there) and
    /// tells the caller to exit. Interactive logins are never session 0, so this can't fire on a
    /// desktop or RDP session. `-softwarerender` skips the block as an escape hatch.
    /// </summary>
    private static bool BlockHiddenServiceSession(string[] args)
    {
        try
        {
            if (args.Any(a => a.Equals("-softwarerender", StringComparison.OrdinalIgnoreCase))) return false;
            if (Process.GetCurrentProcess().SessionId != 0) return false;

            Breadcrumb("session 0 (hidden service session, e.g. NinjaOne Background Mode) — " +
                       "WPF cannot render here; showed the notice and exited");
            const uint MB_OK = 0x0, MB_ICONINFORMATION = 0x40, MB_SETFOREGROUND = 0x10000, MB_TOPMOST = 0x40000;
            MessageBox(IntPtr.Zero,
                "Partner Tool can't display its interface in this session.\n\n" +
                "Background/hidden sessions such as NinjaOne Background Mode can only show classic " +
                "GDI applications — the Partner Tool window would appear as a black rectangle.\n\n" +
                "Connect with full remote control instead to use Partner Tool on this machine.",
                "Partner Tool", MB_OK | MB_ICONINFORMATION | MB_SETFOREGROUND | MB_TOPMOST);
            return true;
        }
        catch { return false; }   // if detection itself fails, let the app try to start normally
    }

    private static void ApplyRenderModeFallback(string[] args)
    {
        try
        {
            bool forced = args.Any(a => a.Equals("-softwarerender", StringComparison.OrdinalIgnoreCase));

            // 0xFFFFFFFF = no console session attached at all (transition state) — treat as hidden.
            uint console = WTSGetActiveConsoleSessionId();
            bool hidden  = console == 0xFFFFFFFF ||
                           (uint)Process.GetCurrentProcess().SessionId != console;

            if (!forced && !hidden) return;   // normal desktop → keep hardware rendering, unchanged

            RenderOptions.ProcessRenderMode = RenderMode.SoftwareOnly;
            SoftwareRendering = true;
            Breadcrumb(forced
                ? "software rendering forced via -softwarerender"
                : $"software rendering enabled — non-console session (session {Process.GetCurrentProcess().SessionId}, console {unchecked((int)console)})");
        }
        catch { /* detection must never block startup; default = normal rendering */ }
    }

    /// <summary>
    /// One startup-diagnostics line in the errors log. Cheap insurance for environments we can't
    /// reproduce locally (Backstage, odd GPUs): if the app "doesn't load", this shows how far it got.
    /// </summary>
    internal static void Breadcrumb(string message)
    {
        try
        {
            const string dir = @"C:\PCI\Logs";
            Directory.CreateDirectory(dir);
            File.AppendAllText(Path.Combine(dir, "PartnerTool_errors.log"),
                LogText.ToAscii($"{DateTime.Now:u} [Startup] {message}{Environment.NewLine}"),
                LogText.Utf8NoBom);
        }
        catch { /* logging must never throw */ }
    }

    private static Assembly? OnAssemblyResolve(object? sender, ResolveEventArgs args)
    {
        // Last-resort resolver (only fires after normal probing fails). Map the failed
        // System.Diagnostics.Tracing request to whatever assembly currently provides EventSource
        // (forwarded to System.Private.CoreLib at runtime), letting type lookups succeed.
        if (new AssemblyName(args.Name).Name?.Equals(
                "System.Diagnostics.Tracing", StringComparison.OrdinalIgnoreCase) == true)
            return typeof(System.Diagnostics.Tracing.EventSource).Assembly;
        return null;
    }

    private void OnUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
    {
        LogCrash("UI", e.Exception);

        // WPF's shutdown telemetry throws FileNotFoundException(System.Diagnostics.Tracing) in
        // single-file builds as the app closes. MainWindow.OnClosed exits the process before that
        // path normally runs, but if it ever fires anyway, swallow it silently — the app is
        // closing, so a "Something went wrong" dialog is wrong and alarming.
        if (e.Exception is FileNotFoundException fnf &&
            (fnf.FileName?.Contains("System.Diagnostics.Tracing", StringComparison.OrdinalIgnoreCase) == true ||
             e.Exception.StackTrace?.Contains("CriticalShutdown") == true))
        {
            e.Handled = true;
            return;
        }

        MessageWindow.Show("Partner Tool", "Something went wrong",
            $"{e.Exception.Message}\n\nThe tool will keep running.",
            MessageKind.Error, MainWindow);
        e.Handled = true;
    }

    private static void LogCrash(string source, Exception? ex)
    {
        try
        {
            const string dir = @"C:\PCI\Logs";
            Directory.CreateDirectory(dir);
            File.AppendAllText(Path.Combine(dir, "PartnerTool_errors.log"),
                $"{DateTime.Now:u} [{source}] {ex}{Environment.NewLine}{Environment.NewLine}");
        }
        catch { /* logging must never throw */ }
    }
}
```
