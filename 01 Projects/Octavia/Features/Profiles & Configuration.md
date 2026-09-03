---
project: Octavia
tags: [octavia, feature]
---

# Profiles & Configuration

*Stage 2, v0.3.0; command-line selection and the flattening fix in v0.4.1; the persistence fix in v0.6.1; **local-first defaults in v0.19.3**.* `<data>\config.json`, read at startup.

## The problem profiles solve

Different machines want **different settings**, and the difference is not small: a weak box wants a small Whisper model, a strong one wants `large-v3-turbo`, and the choice between a free local brain and a paid hosted one is a different axis again. Without profiles that is four keys hand-edited every time — and inevitably one gets forgotten.

## How it works

The file carries all of them, and one flag chooses:

```json
{
  "Profile": "home",
  "Profiles": {
    "home":  { "Brain": "local",  "WhisperModel": "large-v3-turbo" },
    "cloud": { "Brain": "claude", "WhisperModel": "large-v3-turbo" },
    "dev":   { "Brain": "local",  "WhisperModel": "small.en" }
  }
}
```

The named entry is merged over the base settings **in memory only**, so the file keeps describing every machine.

| Profile | Brain | Whisper | For |
|---|---|---|---|
| `home` | local | `large-v3-turbo` | **The default.** Costs nothing, needs no key, works offline |
| `cloud` | Claude | `large-v3-turbo` | When the hosted model is actually wanted |
| `dev` | local | `small.en` | A machine that cannot carry the good speech model |

`live` is still accepted and resolves to what `cloud` does, so an older `config.json` keeps behaving exactly as it did.

## Why local is the base default *(v0.19.3)*

**The base `Brain` is `local`, and that matters more than the profile names.** An unnamed or misspelled profile falls through to the base settings — so before this, a typo in `--profile` produced an assistant pointed at Claude, and on a machine with no key stored that refuses *every single turn* with "No API key yet". She looked broken rather than limited.

This was not hypothetical. The config on the development machine said `Profile: "live"` → `Brain: "claude"` with no key ever stored; she worked only because the desktop shortcut passed `--profile dev`. Started any other way she could not answer at all, and the API-key warning cluttering her face was this, reporting a real misconfiguration correctly.

The local brain needs no key, no network and no account. It is the only honest thing for a fallback path to land on. See [[Lessons Learned]] — *fall back to the thing that needs nothing*.

## Choosing one — three sources, highest wins

| Source | Example | Use |
|---|---|---|
| Command line | `Octavia.exe --profile dev` | A launcher that must not drift |
| Environment | `set OCTAVIA_PROFILE=dev` | One shell session |
| `Profile` key | `"Profile": "dev"` | The machine's normal state |

**Why the command line had to exist.** A Windows shortcut can pass an argument but *cannot* set an environment variable. Before v0.4.1 the only way to pin a launcher's profile was the config file — which the app itself rewrites, so the desktop shortcut's brain depended on a mutable file nobody was watching. `--profile dev` on the shortcut states the intent where the intent belongs. `--profile=dev` and `-p dev` work too.

An unknown profile name falls through to the base settings rather than failing. **Since v0.19.3 that is a warning naming the profiles that do exist** — it used to be an info line, which is precisely how the misconfiguration above sat unnoticed for weeks. Anything that quietly substitutes one configuration for another needs to say so loudly enough to be read.

**Where it shows.** The active profile appears in the face's status panel and the tray tooltip (`Octavia — dev (local)`), and the log names the profile *and the source of the choice*: `profile 'dev' (command line): brain=local, whisper=small.en`.

**She is single-instance**, so `--profile` cannot switch a running Octavia — a second launch surfaces the first. That case is logged rather than passed over in silence.

## The flattening bug (v0.4.1)

`Save()` used to serialise the object it was called on — which, after loading, is the *merged* one. Changing her voice from the console on the `dev` profile therefore wrote `"Brain": "local"` and `"WhisperModel": "small.en"` into the **base** settings, permanently. The profiles still existed but no longer meant anything, and every later run inherited whichever profile happened to be active when a setting was last saved.

The merged copy now keeps the file as it was read, and `Save()` writes runtime changes back into *that*.

**The first version of the fix was itself a bug with a delay on it.** It carried back a hand-kept list of properties — `VoiceName` and `VoiceRate`. Two versions later the settings menu added `AvatarFile` and `RoomHour`, which reached the host, changed the face, wrote a log line, and were silently dropped on save. The list had no way to know they existed.

So `Save()` now keeps a second snapshot: **the merged settings as they stood at load**. Anything that differs from that snapshot now was changed while she was running, and only those keys are written back. It is the same guarantee, computed rather than remembered, and it cannot fall behind a new setting.

`tools\EarsTest` asserts all of it: save on `dev` and the file's base brain is still `claude`; a brand-new setting persists; two saves in a row do not undo each other. See [[Lessons Learned]].

## Keys

Defaults are the ones in `Core\OctaviaConfig.cs`, not the ones in any particular machine's file.

### Brain

| Key | Default | Notes |
|---|---|---|
| `Profile` | `home` | `--profile` then `OCTAVIA_PROFILE` win |
| `Brain` | `local` | `local` or `claude` — **local since v0.19.3** |
| `Model` | `claude-sonnet-5` | Only used when `Brain` is `claude` |
| `LocalEndpoint` | `http://localhost:11434/v1` | Any OpenAI-compatible server |
| `LocalModel` | `llama3.2:3b` | Must be pulled on that server first |
| `MaxTokens` | `1024` | She is told to be brief; this is a backstop |

### Ears

| Key | Default | Notes |
|---|---|---|
| `Recognizer` | `whisper` | `whisper` or `windows` |
| `WhisperModel` | `large-v3-turbo` | |
| `WhisperLanguage` | `en` | or `auto` |
| `WhisperCompute` | `auto` | `auto` / `cpu` / `gpu` — see [[Whisper Integration]] |
| `WhisperThreads` | `0` | 0 lets Whisper choose |
| `RecognitionCulture` | `en-US` | Windows recognizer only |
| `MinConfidence` | `0.35` | Raise if she answers the television |
| `MinUtteranceChars` | `2` | |
| `MicrophoneDevice` | *(empty)* | Substring of the device name; empty is the Windows default |

### Voice

| Key | Default | Notes |
|---|---|---|
| ~~`VoiceEngine`~~, ~~`VoiceName`~~, ~~`NeuralVoiceName`~~ | — | **Deleted in v0.40.0.** She has one voice, chosen by ear out of twenty-two, and there is nothing left for any of the three to say. Old keys in an existing file are simply never read — see [[The Voice]] |
| `VoiceRate` | `0` | −10 to 10, rising is faster. **The only thing about her voice that is still a setting**, because it is about the listener rather than the speaker |

> **They were deleted rather than kept and ignored**, which is the opposite of what happened to `VoiceEngine` when SAPI went — and right for a different reason. That one was kept because a face could still *send* `setVoiceEngine`, and it had to mean something; the answer to that lives in `OctaviaSession` now, where the message is refused out loud. **A field in a config file is not a message from anybody. It is a promise that this can be configured.**

### Her rounds *(v0.42.0–0.44.0)*

| Key | Default | Notes |
|---|---|---|
| `Rounds.Enabled` | `true` | Whether she checks anything on her own — see [[Her Rounds]] |
| `Rounds.EveryMinutes` | `60` | Clamped to 1–1440 |
| `Rounds.LearnForDays` | `7` | She says **nothing at all** until this has passed, counted from her first walk rather than from installation |
| `Rounds.QuietFrom` / `.QuietTo` | `00:00` / `08:30` | Findings inside the window are held, not dropped. Equal values mean no quiet hours. `24:00` is accepted as midnight, because that is what people type |

### Logs

| Key | Default | Notes |
|---|---|---|
| `LogLevel` | `info` | `debug` when reproducing a fault |
| `LogKeepDays` | `14` | **One file per day**; older ones deleted after midnight. `0` keeps everything — see [[Diagnostics]] |

### Attention

| Key | Default | Notes |
|---|---|---|
| `Gate` | `local` | See [[The Attention Gate]] |
| `GateModel` | `llama3.2:3b` | Must be non-reasoning; see [[Lessons Learned]] |
| `GateFollowUpSeconds` | `25` | How long a conversation stays open |
| `WakeNames` | `Octavia` | |

### Senses and the room

| Key | Default | Notes |
|---|---|---|
| `Music` | `true` | Listen to what this machine plays — see [[Music]] |
| `MusicFromRoom` | `false` | Also listen through the microphone (v0.16.0) |
| `OutputDevice` | *(empty)* | Which endpoint the loopback taps |
| `Camera` | `false` | See [[Eyes]] |
| `CameraDevice` | *(empty)* | |
| `AvatarFile` | *(empty)* | A `.vrm` in her avatars folder; empty is the bust |
| `RoomHour` | `-1` | Pin the room to an hour 0–23; negative follows the clock |
| `ShowStats` | `true` | The status readout over her top-left corner (v0.18.0) |

### Host

| Key | Default | Notes |
|---|---|---|
| `FacePort` | `8848` | 0 picks any free port — see [[Face Protocol]] |
| `RemoteAccess` | `false` | Bind beyond loopback, for a face on another device |
| `McpServers` | *(none)* | Tool servers — see [[Roadmap]] stage 12 |
| `Hotkey` | `Ctrl+Alt+O` | Ctrl+Alt+Space is usually taken by an IME |
| `LogLevel` | `info` | |
| `DevPanel` | *(null)* | Null follows the profile; `true`/`false` overrides |
| `ListenOnStart` | `false` | |
| `StartMinimised` | `false` | |

## Data locations

`<data>` is one folder holding everything she owns, resolved by `Core\Paths.cs` at startup:

1. `OCTAVIA_DATA`, if set.
2. **`<repo>\data`** whenever she runs from a build inside the repo — `Paths` walks up from
   the executable looking for `Octavia.slnx`, so `dotnet run`, a Debug shortcut and
   `dist\Octavia.exe` all resolve to the same place.
3. `%APPDATA%\Octavia` otherwise: the installed case, and the only writable choice if she
   ever lives under Program Files.

*Changed 08/30/2026.* It was `%APPDATA%\Octavia` unconditionally until then. Keeping it in
the repo means copying the project copies her models, voice and avatars with it — after a
machine move in which her whole data folder was lost. `data\` is git-ignored: hundreds of
megabytes of downloaded artefacts, none of it source. Note this is a **deliberate
qualification** of the rule in [[The Avatar Interface]] that avatars live outside the
install — that reasoning still holds for an installed copy, which is why case 3 remains.

| Path | What |
|---|---|
| `<data>\config.json` | This file |
| `<data>\apikey.dat` | DPAPI-sealed key — does not travel between machines |
| `<data>\octavia.log` | Startup, profile resolution, errors, and anything the face throws |
| `<data>\models\` | Downloaded Whisper models |
| `<data>\WebView2\` | Browser user data |
| `<data>\avatars\` | VRM characters — see [[The Avatar Interface]] |
| `<data>\voices\` | The neural speech engine and its models — see [[The Voice]] |

`OCTAVIA_CONFIG` points her at a different settings file entirely. It exists so the test harness can exercise loading and saving without touching the real one.

## There is a window for this now *(v0.46.0)*

`Octavia.Control.exe` — see [[Her Controls]] — edits every key above without a text editor, and **it edits the file rather than a running session**, so it works while she is stopped. That is the case that matters: the moment somebody needs to change a setting is usually the moment the setting is stopping her.

> **It says which keys the active profile overrides**, at the top of the window. A value edited there that the profile also sets is written correctly and then ignored at startup, and letting that happen quietly would be the most confusing thing a settings screen could do.

## Notes

The file is written with `UnsafeRelaxedJsonEscaping` so it stays hand-editable — the default encoder escapes `+`, which turns `"Ctrl+Alt+O"` into something that looks corrupted.

A malformed file logs and falls back to defaults rather than refusing to start.
