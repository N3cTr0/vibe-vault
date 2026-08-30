---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Diagnostics\SystemReport.cs
---

# src\Octavia.App\Diagnostics\SystemReport.cs

```csharp
using System.Globalization;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Security.Principal;
using Microsoft.Web.WebView2.Core;
using NAudio.CoreAudioApi;
using Octavia.Core;
using Octavia.Face;
using Octavia.Senses;

namespace Octavia.Diagnostics;

internal sealed record Fact(string Name, string Value);

/// The facts about this machine that a fault report needs and a support conversation
/// otherwise spends three messages establishing.
internal static class SystemReport
{
    public static string Version { get; } =
        Assembly.GetExecutingAssembly()
                .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion
                .Split('+')[0]
        ?? "unknown";

    public static IReadOnlyList<Fact> Gather(OctaviaConfig config, HostSnapshot host)
    {
        var facts = new List<Fact>
        {
            new("Octavia", Version),
            new(".NET", RuntimeInformation.FrameworkDescription),
            new("Windows", $"{RuntimeInformation.OSDescription} ({RuntimeInformation.OSArchitecture})"),
            new("WebView2 runtime", WebViewVersion()),
            new("Elevated", Elevated() ? "yes" : "no"),
            new("Data folder", Paths.DataDir),
            new("Profile", config.Profile),
            new("Brain", host.Brain),
            new("Brain ready", !host.Running ? "not started" : host.BrainReady ? "yes" : "no key stored"),
            new("Recognizer", config.Recognizer),
            new("Speech model", SpeechModel(config)),
            new("Silero VAD", File.Exists(WhisperModelStore.SileroPath) ? "present" : "MISSING"),
            new("CUDA runtime", CudaPresent() ? "present" : "not installed (CPU only)"),
            new("Ears", host.Ears),
            new("Voice", string.IsNullOrEmpty(host.Voice) ? "none selected" : host.Voice),
            new("Voice engine", config.VoiceEngine),
            new("Faces attached", Attachments(host)),
            new("Log level", Log.Threshold.ToString().ToLowerInvariant()),
            new("Errors logged", $"{Log.Errors} error(s), {Log.Warnings} warning(s) this run")
        };

        facts.AddRange(AudioFacts(host));
        return facts;
    }

    private static string Attachments(HostSnapshot host)
    {
        if (!host.Running) return "she was not running when this was taken";

        var faces = host.Faces;
        var socket = faces.SocketBound
            ? $"socket on 127.0.0.1:{faces.Port} with {faces.SocketFaces} attached"
            : "socket NOT BOUND";

        return $"{faces.Attached} total — {(faces.Page ? "built-in page, " : "")}{socket}";
    }

    private static string SpeechModel(OctaviaConfig config)
    {
        var path = WhisperModelStore.PathFor(config.WhisperModel);
        if (!File.Exists(path)) return $"{config.WhisperModel} — NOT DOWNLOADED";

        var mb = new FileInfo(path).Length / 1024d / 1024d;
        return $"{config.WhisperModel} ({mb:0} MB)";
    }

    private static bool CudaPresent() =>
        Directory.EnumerateFiles(AppContext.BaseDirectory, "*cuda*", SearchOption.AllDirectories).Any();

    private static string WebViewVersion()
    {
        try { return CoreWebView2Environment.GetAvailableBrowserVersionString() ?? "not found"; }
        catch (Exception ex) { return $"not found ({ex.Message})"; }
    }

    private static bool Elevated()
    {
        try
        {
            using var identity = WindowsIdentity.GetCurrent();
            return new WindowsPrincipal(identity).IsInRole(WindowsBuiltInRole.Administrator);
        }
        catch { return false; }
    }

    /// The inventory that would have made the RDP microphone obvious in one line
    /// instead of an afternoon: which devices exist, which one Windows considers
    /// default, and whether Windows' own meter sees anything on it.
    private static IEnumerable<Fact> AudioFacts(HostSnapshot host)
    {
        var facts = new List<Fact>();

        try
        {
            using var enumerator = new MMDeviceEnumerator();
            var endpoints = enumerator.EnumerateAudioEndPoints(DataFlow.Capture, DeviceState.Active).ToList();

            MMDevice? standard = null;
            try { standard = enumerator.GetDefaultAudioEndpoint(DataFlow.Capture, Role.Multimedia); }
            catch { /* no default endpoint at all, which the next fact reports */ }

            facts.Add(new Fact("Capture devices", endpoints.Count == 0
                ? "none active"
                : string.Join(", ", endpoints.Select(d =>
                    d.ID == standard?.ID ? $"{d.FriendlyName} (default)" : d.FriendlyName))));

            facts.Add(new Fact("Default capture", standard?.FriendlyName ?? "NONE"));

            // The render endpoint matters now too: it is what she listens to for music,
            // and "she never dances" is usually this line saying NONE.
            facts.Add(new Fact("Default output", Senses.Music.LoopbackListener.DefaultDevice() ?? "NONE"));
            facts.Add(new Fact("Music listening", host.Music));

            if (standard is not null)
                facts.Add(new Fact("Signal on default", $"peak {Peak(standard, TimeSpan.FromMilliseconds(600)).ToString("0.000", CultureInfo.InvariantCulture)}"));

            standard?.Dispose();
            foreach (var device in endpoints) device.Dispose();
        }
        catch (Exception ex)
        {
            facts.Add(new Fact("Capture devices", $"could not enumerate: {ex.Message}"));
        }

        return facts;
    }

    /// Windows' own meter rather than our capture stream, so the answer separates
    /// "no audio is arriving" from "our capture is misconfigured".
    public static float Peak(MMDevice device, TimeSpan window)
    {
        var highest = 0f;
        var until = DateTime.UtcNow + window;
        while (DateTime.UtcNow < until)
        {
            var peak = device.AudioMeterInformation.MasterPeakValue;
            if (peak > highest) highest = peak;
            Thread.Sleep(50);
        }

        return highest;
    }
}

/// What only the running session knows, handed to the report so nothing has to reach
/// back into her while she is working.
internal sealed record HostSnapshot(
    string Brain,
    bool BrainReady,
    string Ears,
    string Voice,
    bool Listening,
    bool FaceBuilt,
    FaceStatus Faces,
    bool Running = true,
    string Music = "off")
{
    /// A bundle taken with `--diagnostics` when she is not up: everything about the
    /// machine still applies, but nothing about a session that never started does.
    public static HostSnapshot Stopped { get; } =
        new("not started", false, "not started", "", false, false, default, Running: false);
}
```
