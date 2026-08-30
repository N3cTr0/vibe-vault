---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Core\OctaviaConfig.cs
---

# src\Octavia.App\Core\OctaviaConfig.cs

```csharp
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Text.Json.Serialization;

namespace Octavia.Core;

internal sealed class OctaviaConfig
{
    /// Which named entry in Profiles to overlay on the settings below. A --profile
    /// argument wins over this, and OCTAVIA_PROFILE sits between the two.
    public string Profile { get; set; } = "live";

    /// Named overrides, so one machine can be a cheap test rig and another the real
    /// thing without hand-editing four keys each time.
    public Dictionary<string, JsonObject> Profiles { get; set; } = new()
    {
        ["dev"] = new JsonObject
        {
            ["Brain"] = "local",
            ["LocalModel"] = "llama3.2:3b",
            ["WhisperModel"] = "small.en",
            ["Recognizer"] = "whisper"
        },
        ["live"] = new JsonObject
        {
            ["Brain"] = "claude",
            ["WhisperModel"] = "large-v3-turbo",
            ["Recognizer"] = "whisper"
        }
    };

    /// Loopback port the face protocol listens on. 0 picks any free port; a fixed
    /// port makes an external face's URL stable across restarts. See PROTOCOL.md.
    public int FacePort { get; set; } = 8848;

    /// "claude" for the real thing, "local" for an OpenAI-compatible server.
    public string Brain { get; set; } = "claude";

    public string Model { get; set; } = "claude-sonnet-5";
    public int MaxTokens { get; set; } = 1024;

    /// Any OpenAI-compatible endpoint: Ollama, LM Studio, llama-server.
    public string LocalEndpoint { get; set; } = "http://localhost:11434/v1";
    /// Wall-clock beats tokens-per-second here: a chattier model that ignores
    /// "be brief" is slower to finish than a slower one that stops talking.
    public string LocalModel { get; set; } = "llama3.2:3b";

    /// "whisper" (local, accurate, downloads a model on first listen) or "windows".
    public string Recognizer { get; set; } = "whisper";

    /// tiny / base / small / medium / large-v3 / large-v3-turbo, with .en variants
    /// for the smaller ones. Turbo is the conversation default; large-v3 when
    /// accuracy beats latency.
    public string WhisperModel { get; set; } = "large-v3-turbo";

    /// ISO code Whisper listens for, or "auto" to detect per utterance.
    public string WhisperLanguage { get; set; } = "en";

    /// Only used by the "windows" recognizer.
    public string RecognitionCulture { get; set; } = "en-US";

    /// "windows" for Windows' own synthesiser, "neural" for Piper. Windows is the
    /// default because it is on every machine and needs nothing downloaded.
    public string VoiceEngine { get; set; } = "windows";

    /// A SAPI voice name, or a Piper voice like "en_GB-jenny_dioco-medium", depending
    /// on the engine above.
    public string? VoiceName { get; set; }

    /// The neural engine's own choice, kept separately so switching engines back and
    /// forth does not lose either selection.
    public string NeuralVoiceName { get; set; } = "en_GB-jenny_dioco-medium";

    public int VoiceRate { get; set; }

    /// A .vrm file in %APPDATA%\Octavia\avatars. Empty means the plaster bust, which
    /// is not a placeholder so much as the fallback that always works. The default is
    /// empty because a fresh machine has no characters on it yet.
    public string AvatarFile { get; set; } = "";

    /// Pins the room's lighting to an hour, 0–23. Negative follows the real clock.
    public int RoomHour { get; set; } = -1;

    /// Whether she listens to what the machine is playing, so she can move to it. The
    /// analysis is local and keeps nothing: a tempo and a loudness, never audio. Off
    /// means the loopback device is not opened at all — which is the setting for a
    /// machine where anything at all might come out of the speakers.
    public bool Music { get; set; } = true;

    /// Whether the face offers its dev panel, which drives every performance she can
    /// give by hand. Null follows the profile — on for `dev`, off for `live` — which is
    /// the right answer often enough that setting this is the exception.
    public bool? DevPanel { get; set; }

    /// Wakes the window and toggles listening from anywhere. Ctrl+Alt+Space is
    /// commonly taken by an IME, so it is not the default.
    public string Hotkey { get; set; } = "Ctrl+Alt+O";
    public bool ListenOnStart { get; set; }
    public bool StartMinimised { get; set; }

    /// debug / info / warn / error. Raise it to debug when reproducing a fault for a
    /// diagnostics bundle; info is what a working machine should be reading.
    public string LogLevel { get; set; } = "info";

    /// Whether she may open the camera when asked to look at something. **Off by
    /// default, and it is the only sense that is** — a microphone in a room is expected,
    /// a camera is not. She never watches: one still, only when a question needs eyes,
    /// and the device is released immediately. See ROADMAP.md stage 9.
    public bool Camera { get; set; }

    /// "local" to judge what she overhears before answering it, "off" to answer
    /// everything the recogniser produces. A room microphone hears the television and
    /// both halves of conversations she is not in; this is what makes leaving her
    /// listening affordable. See ROADMAP.md stage 9.
    public string Gate { get; set; } = "local";

    /// The model that judges. **Keep this the same as LocalModel when the local brain
    /// is in use.** A different one is not merely a second download — the server evicts
    /// one to load the other, and a measured swap cost 24 seconds against 0.7 for a
    /// warm call. A smaller, *separate* gate model is slower than a larger resident one.
    ///
    /// Avoid reasoning models here whatever their size. They spend their whole token
    /// budget deliberating a yes-or-no question and return an empty answer.
    public string GateModel { get; set; } = "llama3.2:3b";

    /// How long after she answers a further remark still counts as the same
    /// conversation, and so skips the gate. Without this her name would have to be
    /// said every single turn.
    public int GateFollowUpSeconds { get; set; } = 25;

    /// What she answers to, comma separated. Anything containing one of these is
    /// always let through, whatever else the gate thinks.
    public string WakeNames { get; set; } = "Octavia";

    /// Utterances shorter than this are treated as noise rather than speech.
    public int MinUtteranceChars { get; set; } = 2;

    /// Recognition confidence below this is discarded instead of sent to the model.
    public float MinConfidence { get; set; } = 0.35f;

    /// Which processor Whisper runs on: "auto", "cpu" or "gpu".
    ///
    /// **"auto" is not neutral — it means GPU.** Whisper.net's own default library
    /// order is [Cuda, Cuda12, Vulkan, CoreML, OpenVino, Cpu, CpuNoAvx], so referencing
    /// the CUDA runtime at all makes any CUDA-capable card win, however slow it is.
    /// A weak card therefore beats a strong CPU purely by being a card. Measure before
    /// believing "auto" chose well — `EarsTest compute` does exactly that.
    public string WhisperCompute { get; set; } = "auto";

    /// Threads Whisper transcribes with. 0 picks half the logical processors, which is
    /// physical-core count on an SMT machine.
    ///
    /// That default is safe rather than optimal. Measured on a Ryzen 7 3700X (8 cores,
    /// 16 threads) with `small.en`: 4 threads 5.55s, 8 threads 4.12s, **16 threads
    /// 3.66s** — so SMT was worth 11% here, against the usual advice that whisper.cpp
    /// gains nothing from it. One machine is not a rule, which is why this is a knob
    /// and not a new default. `EarsTest compute cpu small.en 16` measures your own.
    public int WhisperThreads { get; set; }

    /// Capture device she listens through. Empty follows the Windows default.
    /// Match on the device name as Windows reports it; a substring is enough, so
    /// "Jabra" finds "Headset Microphone (Jabra EVOLVE 20 MS)".
    public string MicrophoneDevice { get; set; } = "";

    /// Render device she listens *to* for music, via loopback. Empty follows the
    /// Windows default — which on a machine with streaming software installed is
    /// often a virtual endpoint that normalises everything to full scale and leaves
    /// no beat to find. Matched like MicrophoneDevice.
    public string OutputDevice { get; set; } = "";

    /// Camera she takes a still from. Empty uses the browser's default. Matched by
    /// substring against the device label.
    public string CameraDevice { get; set; } = "";

    /// The MCP servers that give her hands, keyed by the name their tools are prefixed
    /// with. Empty — the default — means she can talk and nothing else, which is the
    /// right state for a machine that has not been told what it is allowed to touch.
    ///
    /// Each entry is a command she runs and speaks JSON-RPC to over stdio. Home
    /// Assistant, UniFi and anything else are all this shape, which is the point: a new
    /// capability is a new entry here rather than new code in the session.
    ///
    /// **Tokens belong in `Env`, not in a URL**, so they stay out of logs and out of the
    /// diagnostics bundle's redaction problem. See ROADMAP.md stage 12.
    public Dictionary<string, McpServer> McpServers { get; set; } = [];

    private static readonly JsonSerializerOptions Options = new()
    {
        WriteIndented = true,
        Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    /// The file as it was read, before the profile was overlaid. Runtime changes are
    /// written back into this, never into the merged copy.
    private JsonObject? _base;

    /// The merged settings as they stood immediately after loading. Anything that
    /// differs from this now was changed while she was running — which is a far more
    /// reliable definition of "runtime change" than a hand-kept list of properties,
    /// because that list is wrong the moment somebody adds a setting.
    private JsonObject? _loaded;

    public static OctaviaConfig Load(string? requestedProfile = null)
    {
        OctaviaConfig config;
        try
        {
            if (File.Exists(Paths.ConfigFile))
            {
                config = JsonSerializer.Deserialize<OctaviaConfig>(
                    File.ReadAllText(Paths.ConfigFile)) ?? new OctaviaConfig();
            }
            else
            {
                config = new OctaviaConfig();
                config.Save();
            }
        }
        catch (Exception ex)
        {
            Log.Write($"config load failed, using defaults: {ex.Message}");
            config = new OctaviaConfig();
        }

        var applied = config.WithProfileApplied(requestedProfile);
        Log.SetThreshold(applied.LogLevel);
        return applied;
    }

    /// Overlays the selected profile's keys onto the loaded settings. The profile is
    /// applied in memory only — Save writes back to the un-overlaid original, so the
    /// file keeps describing both machines.
    private OctaviaConfig WithProfileApplied(string? requested)
    {
        // A shortcut can pass an argument but cannot set an environment variable,
        // which is why the command line has to outrank both of the others.
        var (wanted, source) =
            !string.IsNullOrWhiteSpace(requested) ? (requested, "command line") :
            Environment.GetEnvironmentVariable("OCTAVIA_PROFILE") is { Length: > 0 } env
                ? (env, "OCTAVIA_PROFILE") : (Profile, "config file");

        if (string.IsNullOrWhiteSpace(wanted)) return this;

        if (!Profiles.TryGetValue(wanted, out var overrides))
        {
            Log.Write($"profile '{wanted}' ({source}) is not defined; using the base settings");
            return this;
        }

        try
        {
            var merged = JsonSerializer.SerializeToNode(this, Options)!.AsObject();
            foreach (var (key, value) in overrides)
            {
                if (key is nameof(Profiles) or nameof(Profile)) continue;
                merged[key] = value?.DeepClone();
            }

            var applied = merged.Deserialize<OctaviaConfig>(Options) ?? this;
            applied.Profile = wanted;
            applied._base = JsonSerializer.SerializeToNode(this, Options)!.AsObject();
            applied._loaded = JsonSerializer.SerializeToNode(applied, Options)!.AsObject();
            Log.Write($"profile '{wanted}' ({source}): brain={applied.Brain}, whisper={applied.WhisperModel}");
            return applied;
        }
        catch (Exception ex)
        {
            Log.Write($"profile '{wanted}' could not be applied: {ex.Message}");
            return this;
        }
    }

    public void Save()
    {
        try
        {
            // Serialising the merged copy would bake the profile overlay into the base
            // settings and the profiles would quietly stop meaning anything. So only the
            // keys that actually changed since load are written back — that is every
            // runtime edit and nothing else, without a list to keep in step.
            if (_base is null || _loaded is null)
            {
                File.WriteAllText(Paths.ConfigFile, JsonSerializer.Serialize(this, Options));
                return;
            }

            var current = JsonSerializer.SerializeToNode(this, Options)!.AsObject();

            foreach (var (key, value) in current)
            {
                if (key is nameof(Profiles) or nameof(Profile)) continue;
                if (JsonNode.DeepEquals(_loaded[key], value)) continue;
                _base[key] = value?.DeepClone();
            }

            File.WriteAllText(Paths.ConfigFile, _base.ToJsonString(Options));

            // What was just written is the new baseline, or the next save would compare
            // against a stale one and think nothing had changed.
            _loaded = current;
        }
        catch (Exception ex)
        {
            Log.Write($"config save failed: {ex.Message}");
        }
    }
}
```
