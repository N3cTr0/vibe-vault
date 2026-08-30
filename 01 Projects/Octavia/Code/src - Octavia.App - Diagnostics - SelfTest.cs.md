---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Diagnostics\SelfTest.cs
---

# src\Octavia.App\Diagnostics\SelfTest.cs

```csharp
using System.Globalization;
using NAudio.CoreAudioApi;
using Octavia.Core;
using Octavia.Senses;
using Octavia.Voice;

namespace Octavia.Diagnostics;

/// <param name="Fix">What to do about it, in a sentence. A red line that does not say
/// what to try is only marginally better than no line.</param>
internal sealed record Check(string Name, bool Ok, string Detail, string? Fix = null);

/// The checks `tools\EarsTest` runs on this machine, but in-app and on demand — because
/// on someone else's PC there is no harness, no debugger, and "it doesn't work" is all
/// that comes back.
///
/// Every check here exists because that exact failure has already happened once.
internal static class SelfTest
{
    public static async Task<IReadOnlyList<Check>> RunAsync(
        OctaviaConfig config, HostSnapshot host, CancellationToken cancel = default)
    {
        var checks = new List<Check> { DataFolder(), Settings(config) };

        // Nothing to say about a transport or a renderer that was never started; a red
        // line for those in a bundle taken while she is stopped would only mislead.
        if (host.Running) checks.AddRange([Transport(host), Renderer(host)]);

        checks.AddRange([Microphone(), SpeechModel(config), Voice(config, host), Avatar(config),
                         Music(config, host), Camera(config), Gate(config)]);

        checks.Add(await BrainAsync(config, host, cancel));
        return checks;
    }

    private static Check DataFolder()
    {
        var probe = Path.Combine(Paths.DataDir, ".writetest");
        try
        {
            File.WriteAllText(probe, "octavia");
            File.Delete(probe);
            return new Check("Data folder", true, Paths.DataDir);
        }
        catch (Exception ex)
        {
            return new Check("Data folder", false, $"{Paths.DataDir} — {ex.Message}",
                "She cannot save settings, models or logs. Check the folder's permissions.");
        }
    }

    private static Check Settings(OctaviaConfig config)
    {
        if (!File.Exists(Paths.ConfigFile))
            return new Check("Settings", false, "config.json is missing",
                "It is written on first run; restarting her will recreate it with defaults.");

        return new Check("Settings", true,
            $"profile '{config.Profile}', brain {config.Brain}, recognizer {config.Recognizer}");
    }

    private static Check Transport(HostSnapshot host)
    {
        var faces = host.Faces;

        if (!faces.SocketBound)
            return new Check("Face transport", false, "the loopback socket did not bind",
                "Another program is probably holding the port. Set \"FacePort\" to 0 in " +
                "config.json to let Windows pick a free one.");

        if (faces.Attached == 0)
            return new Check("Face transport", false, $"listening on 127.0.0.1:{faces.Port}, nothing attached",
                "The renderer is not connected. Restart her; if it persists, send this bundle.");

        return new Check("Face transport", true,
            $"127.0.0.1:{faces.Port}, {faces.Attached} face(s) attached");
    }

    private static Check Renderer(HostSnapshot host) =>
        host.FaceBuilt
            ? new Check("Renderer", true, "WebGL scene built")
            : new Check("Renderer", false, "the scene did not build",
                "The face loaded but three.js failed. Usually a graphics driver without " +
                "WebGL support; the rest of her still works.");

    /// The failure this one is really for: a capture device that opens successfully and
    /// delivers pure digital silence, which looks identical to her ignoring you.
    private static Check Microphone()
    {
        try
        {
            using var enumerator = new MMDeviceEnumerator();
            var endpoints = enumerator.EnumerateAudioEndPoints(DataFlow.Capture, DeviceState.Active).ToList();

            try
            {
                if (endpoints.Count == 0)
                    return new Check("Microphone", false, "no active capture device",
                        "Windows sees no microphone at all. Check it is plugged in and enabled " +
                        "in Sound settings.");

                using var standard = enumerator.GetDefaultAudioEndpoint(DataFlow.Capture, Role.Multimedia);
                var peak = SystemReport.Peak(standard, TimeSpan.FromSeconds(1.2));

                if (peak > 0.005f)
                    return new Check("Microphone", true, $"'{standard.FriendlyName}' carrying signal (peak {peak.ToString("0.000", CultureInfo.InvariantCulture)})");

                return new Check("Microphone", false,
                    $"'{standard.FriendlyName}' is silent (peak {peak.ToString("0.000", CultureInfo.InvariantCulture)})",
                    "Windows itself sees no audio, so this is upstream of Octavia. Speak while " +
                    "running the test; if it stays at 0.000 over Remote Desktop, the client's " +
                    "Local Resources > Remote audio > 'Record from this computer' is off and can " +
                    "only be changed before connecting.");
            }
            finally
            {
                foreach (var device in endpoints) device.Dispose();
            }
        }
        catch (Exception ex)
        {
            return new Check("Microphone", false, ex.Message, "The audio subsystem could not be queried.");
        }
    }

    private static Check SpeechModel(OctaviaConfig config)
    {
        if (!string.Equals(config.Recognizer, "whisper", StringComparison.OrdinalIgnoreCase))
            return new Check("Speech model", true, $"using the Windows recognizer ({config.RecognitionCulture})");

        if (!File.Exists(WhisperModelStore.SileroPath))
            return new Check("Speech model", false, "silero_vad.onnx is missing from Assets",
                "The install is incomplete — without the voice detector she would transcribe " +
                "silence into invented sentences. Reinstall.");

        var path = WhisperModelStore.PathFor(config.WhisperModel);
        if (!File.Exists(path))
            return new Check("Speech model", false, $"'{config.WhisperModel}' has not been downloaded",
                "She fetches it the first time you ask her to listen, which needs internet once.");

        var mb = new FileInfo(path).Length / 1024d / 1024d;
        return new Check("Speech model", true, $"{config.WhisperModel} ({mb:0} MB) present");
    }

    /// "She looks wrong" is a hard bug report to act on; this turns it into a filename.
    private static Check Avatar(OctaviaConfig config)
    {
        if (string.IsNullOrWhiteSpace(config.AvatarFile))
            return new Check("Avatar", true, "the plaster bust (no character configured)");

        var path = Path.Combine(Paths.AvatarDir, config.AvatarFile);
        if (!File.Exists(path))
            return new Check("Avatar", false, $"'{config.AvatarFile}' is not in the avatars folder",
                $"Put the .vrm file in {Paths.AvatarDir}, or clear \"AvatarFile\" to use the bust.");

        var mb = new FileInfo(path).Length / 1024d / 1024d;
        return new Check("Avatar", true, $"{config.AvatarFile} ({mb:0.0} MB)");
    }

    /// Off is reported as a pass, not a warning. A camera that is switched off is a
    /// machine working as configured, and colouring it red would train people to ignore
    /// the panel — or worse, to switch it on to make the red go away.
    private static Check Camera(OctaviaConfig config)
    {
        if (!config.Camera)
            return new Check("Camera", true, "she cannot open a camera (off — the default)");

        return new Check("Camera", true,
            "allowed, one still at a time when a question needs eyes. " +
            "Whether a camera exists is only known when she looks.");
    }

    /// The gate stands between a room microphone and a paid model, so "is it actually
    /// running" is worth a line of its own — silently failing open would be invisible
    /// until the bill arrived.
    private static Check Gate(OctaviaConfig config)
    {
        if (string.Equals(config.Gate, "off", StringComparison.OrdinalIgnoreCase))
            return new Check("Attention gate", true,
                "off — she answers everything she hears");

        if (string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase)
            && !string.Equals(config.GateModel, config.LocalModel, StringComparison.OrdinalIgnoreCase))
            return new Check("Attention gate", false,
                $"judging with '{config.GateModel}' while thinking with '{config.LocalModel}'",
                "Two models cannot both stay loaded, so the server swaps them on every " +
                "utterance — measured at 24 seconds against 0.7 for a warm call. Set " +
                "\"GateModel\" to match \"LocalModel\".");

        return new Check("Attention gate", true, $"{config.GateModel}, {config.GateFollowUpSeconds}s follow-up window");
    }

    /// "She never dances" has three quite different causes — switched off, no output
    /// device, or nothing playing — and they are indistinguishable from the outside.
    private static Check Music(OctaviaConfig config, HostSnapshot host)
    {
        if (!config.Music)
            return new Check("Music", true, "not listening to what the machine plays (switched off)");

        var device = Senses.Music.LoopbackListener.DefaultDevice();
        if (device is null)
            return new Check("Music", false, "no audio output device",
                "Windows has nothing to play through, so there is nothing for her to hear. " +
                "Over Remote Desktop this is normal unless audio is set to play on the remote PC.");

        return host.Running
            ? new Check("Music", true, $"listening to '{device}' — {host.Music}")
            : new Check("Music", true, $"would listen to '{device}'");
    }

    /// Reports the engine she is *actually* speaking with. Checking SAPI while the
    /// neural engine is running would answer a question nobody asked.
    private static Check Voice(OctaviaConfig config, HostSnapshot host)
    {
        if (string.Equals(config.VoiceEngine, "neural", StringComparison.OrdinalIgnoreCase))
        {
            if (!PiperStore.HasEngine)
                return new Check("Voice", false, "the neural speech engine is not installed",
                    "It downloads the first time the neural voice is chosen, which needs internet once. " +
                    "Switch to Windows speech under Settings if that is not possible.");

            if (!PiperStore.HasVoice(config.NeuralVoiceName))
                return new Check("Voice", false, $"'{config.NeuralVoiceName}' has not been downloaded",
                    "Pick it again under Settings > Voice and she will fetch it.");

            // Running means it started; the snapshot carries what the session actually has.
            return new Check("Voice", true, host.Running ? host.Voice : PiperStore.Pretty(config.NeuralVoiceName));
        }

        try
        {
            using var box = new SapiVoice(config);
            var voices = box.InstalledVoices();

            if (voices.Count == 0)
                return new Check("Voice", false, "no speech voices installed",
                    "Add one under Windows Settings > Time & language > Speech.");

            if (!string.IsNullOrWhiteSpace(config.VoiceName) && !voices.Contains(config.VoiceName))
                return new Check("Voice", false, $"'{config.VoiceName}' is not installed on this machine",
                    $"She fell back to '{box.CurrentVoice}'. Pick one from the dropdown to make it stick.");

            return new Check("Voice", true, $"{box.CurrentVoice} ({voices.Count} installed)");
        }
        catch (Exception ex)
        {
            return new Check("Voice", false, ex.Message, "Speech synthesis could not start.");
        }
    }

    /// Deliberately free. The local brain is asked whether it is there; Claude is not
    /// called at all, because a self-test that quietly spends money is a bad self-test.
    private static async Task<Check> BrainAsync(OctaviaConfig config, HostSnapshot host, CancellationToken cancel)
    {
        if (!string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase))
        {
            return host.BrainReady
                ? new Check("Brain", true, $"{host.Brain}, key stored (not called — this test costs nothing)")
                : new Check("Brain", false, "no API key stored",
                    "Paste a key into the field at the bottom of her window. It is sealed to this " +
                    "Windows account and never leaves the machine.");
        }

        try
        {
            using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(4) };
            using var response = await http.GetAsync($"{config.LocalEndpoint.TrimEnd('/')}/models", cancel);

            if (!response.IsSuccessStatusCode)
                return new Check("Brain", false, $"{config.LocalEndpoint} answered {(int)response.StatusCode}",
                    "The server is running but did not answer as an OpenAI-compatible endpoint.");

            var body = await response.Content.ReadAsStringAsync(cancel);
            var listed = body.Contains($"\"{config.LocalModel}\"", StringComparison.OrdinalIgnoreCase)
                      || body.Contains(config.LocalModel, StringComparison.OrdinalIgnoreCase);

            return listed
                ? new Check("Brain", true, $"{config.LocalModel} available at {config.LocalEndpoint}")
                : new Check("Brain", false, $"{config.LocalEndpoint} is up but does not list '{config.LocalModel}'",
                    $"Pull it first: ollama pull {config.LocalModel}");
        }
        catch (Exception ex)
        {
            return new Check("Brain", false, $"no answer from {config.LocalEndpoint} ({ex.Message})",
                "Start the local model server (Ollama or LM Studio), or switch to the 'live' " +
                "profile to use Claude.");
        }
    }
}
```
