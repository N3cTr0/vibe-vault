---
project: Octavia
tags: [octavia, feature]
---

# Profiles & Configuration

*Stage 2, v0.3.0; command-line selection and the flattening fix in v0.4.1; the persistence fix in v0.6.1.* `<data>\config.json`, read at startup.

## The problem profiles solve

The dev VM and the target machine want **opposite settings**. The VM has two cores and no GPU: it wants a small Whisper model and a free local brain. The target box has a modern GPU: it wants `large-v3-turbo` and Claude. Without profiles that is four keys hand-edited every time you move between machines — and inevitably one gets forgotten.

## How it works

The file carries both, and one flag chooses:

```json
{
  "Profile": "dev",
  "Profiles": {
    "dev":  { "Brain": "local", "LocalModel": "llama3.2:3b", "WhisperModel": "small.en" },
    "live": { "Brain": "claude", "WhisperModel": "large-v3-turbo" }
  }
}
```

The named entry is merged over the base settings **in memory only**, so the file keeps describing both machines.

## Choosing one — three sources, highest wins

| Source | Example | Use |
|---|---|---|
| Command line | `Octavia.exe --profile dev` | A launcher that must not drift |
| Environment | `set OCTAVIA_PROFILE=dev` | One shell session |
| `Profile` key | `"Profile": "dev"` | The machine's normal state |

**Why the command line had to exist.** A Windows shortcut can pass an argument but *cannot* set an environment variable. Before v0.4.1 the only way to pin a launcher's profile was the config file — which the app itself rewrites, so the desktop shortcut's brain depended on a mutable file nobody was watching. `--profile dev` on the shortcut states the intent where the intent belongs. `--profile=dev` and `-p dev` work too.

An unknown profile name logs a warning and falls through to the base settings rather than failing.

**Where it shows.** The active profile appears in the face's status panel and the tray tooltip (`Octavia — dev (local)`), and the log names the profile *and the source of the choice*: `profile 'dev' (command line): brain=local, whisper=small.en`.

**She is single-instance**, so `--profile` cannot switch a running Octavia — a second launch surfaces the first. That case is logged rather than passed over in silence.

## The flattening bug (v0.4.1)

`Save()` used to serialise the object it was called on — which, after loading, is the *merged* one. Changing her voice from the console on the `dev` profile therefore wrote `"Brain": "local"` and `"WhisperModel": "small.en"` into the **base** settings, permanently. The profiles still existed but no longer meant anything, and every later run inherited whichever profile happened to be active when a setting was last saved.

The merged copy now keeps the file as it was read, and `Save()` writes runtime changes back into *that*.

**The first version of the fix was itself a bug with a delay on it.** It carried back a hand-kept list of properties — `VoiceName` and `VoiceRate`. Two versions later the settings menu added `AvatarFile` and `RoomHour`, which reached the host, changed the face, wrote a log line, and were silently dropped on save. The list had no way to know they existed.

So `Save()` now keeps a second snapshot: **the merged settings as they stood at load**. Anything that differs from that snapshot now was changed while she was running, and only those keys are written back. It is the same guarantee, computed rather than remembered, and it cannot fall behind a new setting.

`tools\EarsTest` asserts all of it: save on `dev` and the file's base brain is still `claude`; a brand-new setting persists; two saves in a row do not undo each other. See [[Lessons Learned]].

## Keys

| Key | Default | Notes |
|---|---|---|
| `Profile` | `live` | `--profile` then `OCTAVIA_PROFILE` win |
| `Brain` | `claude` | `claude` or `local` |
| `Model` | `claude-sonnet-5` | |
| `LocalEndpoint` | `http://localhost:11434/v1` | Any OpenAI-compatible server |
| `LocalModel` | `llama3.2:3b` | Must be pulled on that server first |
| `MaxTokens` | `1024` | She is told to be brief; this is a backstop |
| `Recognizer` | `whisper` | `whisper` or `windows` |
| `WhisperModel` | `large-v3-turbo` | |
| `WhisperLanguage` | `en` | or `auto` |
| `RecognitionCulture` | `en-US` | Windows recognizer only |
| `VoiceEngine` | `windows` | `windows` or `neural` — see [[The Voice]] |
| `VoiceName` | first installed | The Windows voice; Settings → Voice |
| `NeuralVoiceName` | `en_GB-jenny_dioco-medium` | The Piper voice, kept separately |
| `AvatarFile` | *(empty)* | A `.vrm` in her avatars folder; empty is the bust |
| `RoomHour` | `-1` | Pin the room to an hour 0–23; negative follows the clock |
| `VoiceRate` | `0` | −10 to 10 |
| `Hotkey` | `Ctrl+Alt+O` | Ctrl+Alt+Space is usually taken by an IME |
| `MinConfidence` | `0.35` | Raise if she answers the television |
| `MinUtteranceChars` | `2` | |
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

## Notes

The file is written with `UnsafeRelaxedJsonEscaping` so it stays hand-editable — the default encoder escapes `+`, which turns `"Ctrl+Alt+O"` into something that looks corrupted.

A malformed file logs and falls back to defaults rather than refusing to start.
