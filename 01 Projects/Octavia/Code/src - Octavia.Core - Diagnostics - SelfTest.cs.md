---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Diagnostics\SelfTest.cs
---

# src\Octavia.Core\Diagnostics\SelfTest.cs

```csharp
using System.Net.Http.Json;
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


        checks.AddRange([SpeechModel(config), Voice(config, host), Avatar(config),
                         Music(config, host), Camera(config), Gate(config)]);

        checks.Add(await BrainAsync(config, host, cancel));

        // Only meaningful for a local brain; a cloud brain has no placement to get wrong.
        if (string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase))
            checks.Add(await GateSpeedAsync(config, cancel));
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

    /* **The microphone and music checks are gone** — Stage 15 item 3.

       Both asked Windows about devices on the machine she runs on, and both were good
       checks: the microphone one existed for a capture device that opens successfully and
       delivers pure digital silence, which looks identical to her ignoring you.

       She has no capture device now, and no loopback. The equivalent failures have moved to
       the face that owns the microphone — a browser reports its own permission and its own
       silence — and a self-test that reported on the *server's* microphone would be answering
       a question about the wrong machine, confidently. See `MusicSummary`, which says who
       last reported and how long ago rather than which endpoint was tapped. */

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

        /* Two different models used to be a hard failure: on the 16 GB dev VM the server
           could hold only one, so it swapped on every utterance and a measured warm call
           went from 0.7 s to 24 s.

           **That is a property of the machine, not of the design.** Re-measured on
           08/31/2026 with 32 GB: a 3B gate and a 7B brain both stay resident, alternating
           calls run at 0.39 s and 3.1 s with no swap at all, and 15.9 GB is still free.
           So this is a note now rather than a failure — and a *useful* split, because the
           gate wants speed and the brain wants sense, and no single model is best at both. */
        if (string.Equals(config.Brain, "local", StringComparison.OrdinalIgnoreCase)
            && !string.Equals(config.GateModel, config.LocalModel, StringComparison.OrdinalIgnoreCase))
            return new Check("Attention gate", true,
                $"judging with '{config.GateModel}', thinking with '{config.LocalModel}' — " +
                "check `ollama ps` lists both, or every utterance pays a model swap");

        return new Check("Attention gate", true, $"{config.GateModel}, {config.GateFollowUpSeconds}s follow-up window");
    }

    /// How long the local model actually takes, which no amount of reading config can
    /// tell you.
    ///
    /// This exists because of a failure that was invisible for weeks: Ollama offloaded
    /// the model onto a **GeForce GT 730 over Vulkan** — 28 of 29 layers — so every gate
    /// call took about 3.9 s against 0.63 s on the CPU beside it, and with an 8 s
    /// timeout in front the gate simply failed open on everything. She answered the
    /// television, and the only visible symptom was that she felt slow. Nothing in the
    /// config was wrong; the placement was. See ROADMAP.md.
    private static async Task<Check> GateSpeedAsync(OctaviaConfig config, CancellationToken cancel)
    {
        if (string.Equals(config.Gate, "off", StringComparison.OrdinalIgnoreCase))
            return new Check("Gate speed", true, "gate is off");

        try
        {
            using var http = new HttpClient
            {
                BaseAddress = new Uri(config.LocalEndpoint.TrimEnd('/') + "/"),
                Timeout = TimeSpan.FromSeconds(30)
            };

            var request = new
            {
                model = config.GateModel,
                messages = new object[] { new { role = "user", content = "Reply with the single word OK." } },
                stream = false,
                temperature = 0,
                max_tokens = 8
            };

            // Twice: the first call may load the model, which is not what an utterance pays.
            var stopwatch = System.Diagnostics.Stopwatch.StartNew();
            using (await http.PostAsJsonAsync("chat/completions", request, cancel)) { }
            stopwatch.Restart();
            using var second = await http.PostAsJsonAsync("chat/completions", request, cancel);
            stopwatch.Stop();

            if (!second.IsSuccessStatusCode)
                return new Check("Gate speed", false, $"the local server returned {(int)second.StatusCode}",
                    $"Is '{config.GateModel}' pulled? Try: ollama pull {config.GateModel}");

            var ms = stopwatch.ElapsedMilliseconds;
            if (ms < 1500)
                return new Check("Gate speed", true, $"{ms} ms warm");

            return new Check("Gate speed", false, $"{ms} ms warm — slow enough to time out",
                "The model is probably running on a weak GPU rather than the processor. " +
                "Check with 'ollama ps': anything other than 100% CPU on a machine whose " +
                "processor is the stronger half is costing you. Pin it to the CPU with a " +
                "Modelfile containing 'PARAMETER num_gpu 0', create it as a new name, and " +
                "point LocalModel and GateModel at that.");
        }
        catch (Exception ex)
        {
            return new Check("Gate speed", false, ex.Message,
                "The local model server did not answer. Is it running?");
        }
    }

    /// "She never dances" has three quite different causes — switched off, no output
    /// device, or nothing playing — and they are indistinguishable from the outside.
    private static Check Music(OctaviaConfig config, HostSnapshot host)
    {
        if (!config.Music)
            return new Check("Music", true, "not listening to what the machine plays (switched off)");

        /* This used to name the output endpoint she had tapped, because *"she will not
           dance"* was almost never the analyser and almost always the music going to a
           different endpoint than the one she opened — loopback hears one output and a
           machine has four.

           **She taps nothing now.** The advice moves rather than disappearing: the equivalent
           mystery is that *nothing is reporting*, so `host.Music` says who last told her and
           how long ago. A stale report is left standing on purpose — a client that closed its
           lid has stopped telling her things, which is not the same as telling her the music
           stopped. */
        return new Check("Music", true, host.Music,
            "She is told what is playing rather than hearing it. If she is not moving to " +
            "something, check that a client capable of reporting it is running and connected.");
    }

    /// Reports the engine she is *actually* speaking with. Checking SAPI while the
    /// neural engine is running would answer a question nobody asked.
    private static Check Voice(OctaviaConfig config, HostSnapshot host)
    {
        /* **One voice to check now.** The Windows branch this replaces asked `SapiVoice`
           what speech voices Windows had installed — a sensible question while she could use
           them, and a meaningless one since Stage 15 item 3: SAPI synthesises to a sound
           card, the server has none, and a voice that cannot be streamed reaches nobody.

           The advice changed with it. "Switch to Windows speech under Settings" was the
           fallback for a machine that could not download the model, and there is no longer
           anywhere to fall back to — so it says what is actually true instead. */
        if (!PiperStore.HasEngine)
            return new Check("Voice", false, "the neural speech engine is not installed",
                "It downloads the first time she starts, which needs internet once. " +
                "Until it arrives she has no voice at all — there is no longer a Windows one.");

        if (!PiperStore.HasVoice(config.NeuralVoiceName))
            return new Check("Voice", false, $"'{config.NeuralVoiceName}' has not been downloaded",
                "Pick it again under Settings > Voice and she will fetch it.");

        // Running means it started; the snapshot carries what the session actually has.
        return new Check("Voice", true, host.Running ? host.Voice : PiperStore.Pretty(config.NeuralVoiceName));
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
