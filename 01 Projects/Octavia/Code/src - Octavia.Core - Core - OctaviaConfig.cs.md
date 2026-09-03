---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\OctaviaConfig.cs
---

# src\Octavia.Core\Core\OctaviaConfig.cs

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
    public string Profile { get; set; } = "home";

    /// Named overrides, so one machine can be a cheap test rig and another the real
    /// thing without hand-editing four keys each time.
    ///
    /// `home` is the intended primary and the default: she runs on the local model, which
    /// costs nothing, needs no key and works with the network down. `cloud` is for the
    /// times Claude is actually wanted. `dev` is `home` with a smaller Whisper, for a
    /// machine that cannot carry the good one.
    ///
    /// `live` is kept because existing config files name it. It resolves to the same
    /// thing `cloud` does, so an older file keeps behaving exactly as it did.
    public Dictionary<string, JsonObject> Profiles { get; set; } = new()
    {
        ["home"] = new JsonObject
        {
            ["Brain"] = "local",
            ["WhisperModel"] = "large-v3-turbo",
            ["Recognizer"] = "whisper"
        },
        ["cloud"] = new JsonObject
        {
            ["Brain"] = "claude",
            ["WhisperModel"] = "large-v3-turbo",
            ["Recognizer"] = "whisper"
        },
        ["dev"] = new JsonObject
        {
            ["Brain"] = "local",
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

    /// "local" for an OpenAI-compatible server, "claude" for the hosted model.
    ///
    /// Local is the base default deliberately. An unnamed or misspelled profile falls
    /// back to these settings, and falling back to a brain that refuses every turn
    /// without an API key is the worst of the available failures — she looks broken
    /// rather than limited. Local needs nothing and works offline.
    public string Brain { get; set; } = "local";

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

    /* **Which voice she has is no longer a setting.**

       `VoiceEngine`, `VoiceName` and `NeuralVoiceName` were here: an engine to pick, and a
       voice within it. Stage 16 auditioned twenty-two and the owner chose one, so there is
       nothing left for any of the three to say. They are **deleted rather than kept and
       ignored**, which is the opposite of what happened to `VoiceEngine` when SAPI went —
       and the right call for a different reason. That one was kept because a face could
       still *send* `setVoiceEngine`, and it had to mean something; the answer to that lives
       in `OctaviaSession` now, where the message is refused out loud. A field in a config
       file is not a message from anybody. It is a promise that this can be configured.

       Old keys left in an existing `config.json` are simply not read: nothing here is
       strict about unmapped members, so a file that still says `"NeuralVoiceName"` loads
       exactly as before and quietly stops meaning anything.

       See `KokoroStore`, where the choice that replaced them is written down. */

    /// How fast she speaks: -10 to 10, rising is faster. The one thing about her voice that
    /// is still a preference, because it is about the listener rather than the speaker.
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

    /// Whether she also listens for music **in the room**, through the microphone,
    /// rather than only to what this computer is playing.
    ///
    /// `Music` above is WASAPI loopback: it hears this machine's output and nothing else,
    /// so a speaker across the room is silence to it. This is the other half, and it costs
    /// nothing extra to capture — the microphone is already open when she is listening,
    /// and these are the same frames the voice detector sees.
    ///
    /// **Off by default, and only active while she is listening.** Two honest caveats: a
    /// boom mic in a reverberant room delivers far worse dynamics than a loopback, so the
    /// tempo will be less certain; and it only works when her ears are on, which is a
    /// different condition from `Music` and will surprise someone eventually.
    public bool MusicFromRoom { get; set; }

    /// Whether the status readout — voice, ears, brain, music, profile — is shown over
    /// her top-left corner. On by default because it answers most of the questions
    /// someone asks about her; off is the setting for actually looking at her.
    public bool ShowStats { get; set; } = true;

    /// Whether the face offers its dev panel, which drives every performance she can
    /// give by hand. Null follows the profile — on for `dev`, off for `live` — which is
    /// the right answer often enough that setting this is the exception.
    public bool? DevPanel { get; set; }

    /// Wakes the window and toggles listening from anywhere. Ctrl+Alt+Space is
    /// commonly taken by an IME, so it is not the default.
    public string Hotkey { get; set; } = "Ctrl+Alt+O";
    public bool ListenOnStart { get; set; }
    public bool StartMinimised { get; set; }

    /// Load the speech models when the server starts, rather than when somebody first asks
    /// to be heard. **On by default**, and it is not merely a convenience.
    ///
    /// Opening Whisper takes about three seconds with `large-v3-turbo` on a CPU, and a face
    /// that holds its microphone button begins streaming immediately — so the first press of
    /// a cold session lost its opening words, and lost them *silently*, producing a plausible
    /// truncated sentence rather than an obvious nothing. See ROADMAP.md stage 14 item 11.
    ///
    /// Paying that cost at startup, where nobody is talking, removes the window rather than
    /// papering over it: with the recogniser already open, taking the floor never awaits.
    ///
    /// **This does not open a microphone.** Constructing the recogniser builds the voice
    /// detector and the transcriber and touches no device; `Start` is what opens one, and
    /// that still happens only when somebody listens or holds a button. Turn it off on a
    /// machine where the memory matters more than the first sentence does.
    public bool OpenEarsOnStart { get; set; } = true;

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

    /// The model that judges — and it **should not** be the same as LocalModel any more.
    ///
    /// It used to have to be. On the 16 GB dev VM the server could hold one model, so a
    /// separate gate meant a swap on every utterance: 24 seconds against 0.7 for a warm
    /// call. Re-measured on 32 GB (08/31/2026): a 3B gate and a 7B brain both stay
    /// resident and alternate at 0.39 s and 3.1 s with no swap at all.
    ///
    /// So split them, because the two jobs want opposite things. Measured here, CPU-only:
    ///
    /// | model | gate | correct | reply | tools |
    /// |---|---|---|---|---|
    /// | `llama3.2:3b` | 688 ms | **4/4** | 4.9 s | 4/4 |
    /// | `qwen2.5:3b`  | 308 ms | 2/4 | 1.6 s | 4/4 |
    /// | `qwen2.5:7b`  | 815 ms | 2/4 | 5.1 s | 4/4 |
    /// | `llama3.1:8b` | 1843 ms | 2/4 | 9.5 s | 4/4 |
    ///
    /// **Bigger was not better at anything measured.** Both Qwens answer NO to "what is
    /// the weather doing tomorrow" — they fail open in the wrong direction, and a gate
    /// that never opens is worse than none. That looks like prompt sensitivity rather
    /// than capability, and is worth chasing: qwen2.5:3b at 308 ms would be the better
    /// gate if the instruction can be made to land. `EarsTest models <name>...` measures it.
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

    /// The spoken phrase that opens an exchange, or empty for none.
    ///
    /// **Not the same thing as `WakeNames` above, and the difference is where the work
    /// happens.** `WakeNames` is matched in a *transcript* — free, but only after Whisper has
    /// turned every utterance in the room into words. This is matched in the *audio*, by a
    /// four-megabyte model, so nothing is transcribed until she is addressed.
    ///
    /// Empty by default, because hers does not exist yet: openWakeWord ships `hey jarvis`,
    /// `alexa`, `hey mycroft` and `hey rhasspy`, and *"Hey Octavia"* has to be trained. Any
    /// other value is looked up as a file — `Hey Octavia` → `hey_octavia.onnx` in
    /// `data\models\wake`.
    ///
    /// **Two words beat one.** A phrase has a far stronger acoustic signature than a single
    /// name, and it was the largest factor in model quality across every configuration
    /// openWakeWord's own trainers tested.
    public string WakePhrase { get; set; } = "";

    /// How sure the model must be, 0 to 1. openWakeWord's own guidance is 0.5: higher misses
    /// her, lower wakes on the television.
    public double WakeThreshold { get; set; } = 0.5;

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

    /// Whether a face on another machine may connect — a phone, a wall tablet.
    ///
    /// **Off by default and worth leaving off until the network part is done.** On, the
    /// face socket binds every interface instead of loopback, and a remote face must
    /// present the durable key in `remote.key` rather than the per-run token. That key is
    /// one shared secret in front of a microphone and, later, a house: it is enough
    /// behind Tailscale or Wireguard, and it is *not* enough behind a forwarded port.
    /// See ROADMAP.md stage 13.
    public bool RemoteAccess { get; set; }

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
            // A warning, not a note. A misspelled profile silently falls back to the base
            // settings, which is how a machine ends up running something nobody chose.
            Log.Warn($"profile '{wanted}' ({source}) is not defined; using the base settings " +
                     $"(brain={Brain}, whisper={WhisperModel}). Defined: {string.Join(", ", Profiles.Keys)}");
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
