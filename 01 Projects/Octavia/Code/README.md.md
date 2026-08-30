---
project: Octavia
tags: [octavia, code]
source-path: README.md
---

# README.md

```markdown
# Octavia

A talking bust with Claude behind her. A .NET desktop host owns the microphone, the
voice, the API key and the conversation; a WebView2 renders the face and does nothing
else.

## Why it is split this way

The face is a web page because that is the fastest thing to iterate on, and because the
bust should be replaceable — a rigged glTF head, or a panel on a wall tablet — without
touching anything that thinks. Everything a browser structurally cannot do lives in the
host instead: an always-on process, a tray icon, a global hotkey, system audio capture,
and an API key that never reaches the page.

```
Octavia.exe (WPF)
├── Brain/       Claude or a local model, streamed a sentence at a time
├── Senses/      microphone level (NAudio) + VAD + speech recognition
│   └── Music/   WASAPI loopback and the beat detection it feeds
├── Voice/       Windows speech or a neural engine, behind IVoice
├── Audio/       FFT and the lip sync read out of the waveform
├── Diagnostics/ self-test, system report, and the bundle you can send
├── Face/        the message channel every renderer speaks
└── wwwroot/     the renderer: room, avatar, and console
```

Every subsystem sits behind an interface — `IBrain`, `ISpeechRecognizer`, `IVoice`,
`IFaceTransport` — so each is a swap rather than a rewrite. Host and face speak JSON over
a loopback WebSocket (see `PROTOCOL.md`), which is why *anything* that speaks it is a
legal face: the built-in page, a browser on a wall tablet, or an Unreal application later.

## Running her

```bash
dotnet run --project src/Octavia.App
```

On first launch there is no API key. The console at the bottom of the window shows an
API key field — paste an `sk-ant-...` key and press Store. It is sealed with DPAPI to
your Windows account under `%APPDATA%\Octavia\apikey.dat` and is never sent back to the
page. `ANTHROPIC_API_KEY` is used instead when it is set.

- **Microphone button**, `Space`, or the tray menu toggles listening.
- **Ctrl+Alt+O** wakes the window and toggles listening from anywhere.
- **Esc** or **Hush** stops her mid-sentence.
- **Log** opens the transcript; **Forget** clears her memory of the conversation.
- Closing the window hides her to the tray. Quit from the tray menu.

## Moving her to another PC

Build a portable copy that carries its own .NET:

```
rmdir /s /q C:\Projects\Octavia\dist
dotnet publish src\Octavia.App -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:PublishReadyToRun=true -o C:\Projects\Octavia\dist
```

The `rmdir` is not optional politeness: publish overwrites but never deletes, so a
renamed or dropped file lives on in `dist` forever. A stale `lib\three.min.js` survived
the whole three.js upgrade that way.

Copy the whole `dist` folder — `Octavia.exe` and the `wwwroot` beside it. The face is
not embedded in the exe on purpose: you can edit the bust on the target machine and just
reload. About 310 MB.

If the target already has the **.NET 10 Desktop Runtime**, drop `--self-contained true`
and the two `PublishSingleFile` switches for a few MB instead.

Do **not** add `PublishTrimmed`. WPF, System.Speech and the JSON serialiser all resolve
types by reflection, and trimming removes them.

**The API key does not travel.** It is DPAPI-sealed to one Windows account on one
machine, so `apikey.dat` is unreadable anywhere else. Do not copy it — leave it behind
and paste the key in again on the new machine. She handles this gracefully: the read
fails, and she asks for a key as though she were new. Config, log and downloaded models
live in `%APPDATA%\Octavia` too, and are regenerated with defaults.

The target also needs the **WebView2 runtime** (present on Windows 11 by default; she
shows a message naming it if it is missing). For the Windows recognizer fallback it also
needs a speech recognizer for your language, under Settings, Time and language, Speech.

## Two brains

`Brain: "claude"` is her real mind. `Brain: "local"` points her at any server speaking
the OpenAI-compatible chat API — Ollama, LM Studio, `llama-server` — so development
costs nothing and works offline:

```
ollama serve
ollama pull llama3.2:3b
```

A 3-4B model **will not hold her persona** — expect rambling no matter what the system
prompt says. It is a test of the pipeline, not of her character. Her `<think>` blocks
are stripped before they reach the voice, and markdown is flattened so asterisks are
never read aloud.

Pick on wall clock, not tokens per second: a chattier model that ignores "be brief" is
slower to finish than a slower one that stops talking.

## Profiles

The dev machine and the real one want opposite settings, so `config.json` carries both
and one flag chooses. `dev` selects the local brain and `small.en`; `live` selects
Claude and `large-v3-turbo`.

Three ways to pick one, highest wins:

```
Octavia.exe --profile dev
set OCTAVIA_PROFILE=dev && dotnet run --project src/Octavia.App
```

...and the `Profile` key in the file if neither is given. **A desktop shortcut can pass
an argument but cannot set an environment variable, which is why `--profile` exists** —
a launcher that names its profile cannot drift when the config file changes. The tray
tooltip and the face's status panel both show which one is live, and the log records
the profile *and where the choice came from*.

Profiles are applied in memory only — saving carries runtime changes back to the
un-overlaid settings, so the file keeps describing both machines. Add your own by
editing the `Profiles` block.

She is single-instance: launching a second copy surfaces the first rather than starting
a new one, so `--profile` cannot switch a running Octavia. Quit her from the tray first.

## Configuration

`%APPDATA%\Octavia\config.json`, re-read at startup:

| Key | Default | Notes |
|---|---|---|
| `Profile` | `live` | Which `Profiles` entry to overlay; `--profile` then `OCTAVIA_PROFILE` win |
| `Brain` | `claude` | `claude` or `local` |
| `Model` | `claude-sonnet-5` | Cheaper and newer than Sonnet 4.6 |
| `LocalEndpoint` | `http://localhost:11434/v1` | Ollama's default; any OpenAI-compatible server works |
| `LocalModel` | `llama3.2:3b` | Must be pulled on that server first |
| `MaxTokens` | `1024` | She is told to be brief; this is a backstop |
| `Recognizer` | `whisper` | `whisper` (local, accurate) or `windows` (instant, mediocre) |
| `WhisperModel` | `large-v3-turbo` | `large-v3` when accuracy beats latency; `small.en` on weak CPUs |
| `WhisperLanguage` | `en` | ISO code, or `auto` to detect per utterance |
| `RecognitionCulture` | `en-US` | Windows recognizer only |
| `VoiceEngine` | `windows` | `windows` or `neural`. Settings → Speech |
| `VoiceName` | first installed | The Windows voice. Settings → Voice |
| `NeuralVoiceName` | `en_GB-jenny_dioco-medium` | The Piper voice, kept separately so switching engines loses neither |
| `VoiceRate` | `0` | -10 to 10 |
| `Hotkey` | `Ctrl+Alt+O` | Ctrl+Alt+Space is usually taken by an IME |
| `MinConfidence` | `0.35` | Raise it if she answers the television, lower it if she ignores you |
| `MinUtteranceChars` | `2` | Shorter transcripts are treated as noise |
| `ListenOnStart` | `false` | See the cost note below |
| `StartMinimised` | `false` | Start hidden in the tray |
| `AvatarFile` | *(empty)* | A `.vrm` in `%APPDATA%\Octavia\avatars`; empty means the bust. Settings → Appearance |
| `RoomHour` | `-1` | Pin the room's lighting to an hour 0–23; negative follows the clock |
| `Music` | `true` | Whether she hears what the machine plays. Off closes the device |
| `Gate` | `local` | `local` judges what she overhears; `off` answers everything |
| `GateModel` | `llama3.2:3b` | **Keep the same as `LocalModel`** — a different one reloads per utterance |
| `GateFollowUpSeconds` | `25` | How long after answering a remark still counts as the same conversation |
| `WakeNames` | `Octavia` | Comma separated. Always let through, whatever the gate thinks |
| `Camera` | `false` | Whether she may open a camera when a question needs eyes |
| `DevPanel` | *(unset)* | Follows the profile — on for `dev`, off for `live` |
| `LogLevel` | `info` | `debug` when reproducing a fault; `warn` / `error` to quieten her |
| `FacePort` | `8848` | `0` picks any free port |

## Her face

The renderer is three.js r180 as ES modules, vendored under `wwwroot\lib` with bare
import specifiers rewritten so no import map is needed and the CSP stays `script-src
'self'`.

Everything she can be driven to do goes through one small avatar interface —
`setViseme`, `setExpression`, `setGaze`, `setBlink`, `setPose`, `update`. The plaster
bust implements it and so does a VRM character. `face.js` owns the *performance* (blink
schedules, saccades, head carriage, mood); the avatar owns how a jaw actually moves.

**To give her a character:** put a `.vrm` file in `%APPDATA%\Octavia\avatars`, then pick
it under **Settings → Appearance**. The menu lists whatever is in the folder, plus the
plaster bust; the choice applies immediately and is saved.

It is a config key too, if you prefer:

```json
{ "AvatarFile": "octavia.vrm" }
```

Leave it empty for the bust. The host maps that folder to a read-only
`https://octavia.avatar` origin — the face never reads arbitrary paths — and offers the
URL in `hello`. If the file is missing or will not load, she stays a bust and says so in
the log rather than showing nothing.

VRM models come from **VRoid Studio** (free, design her yourself), a commission, or any
VRM marketplace. Arms are posed out of the format's T-pose on load, since VRM defines a
rest position rather than an idle.

**The expression vocabulary is VRM 1.0's** — `happy / angry / sad / relaxed / surprised /
neutral`, and the visemes `aa / ih / ou / ee / oh`. That is deliberate: the protocol maps
onto a real character by identity, with no translation layer in between to get wrong.

Her expression is read from the text of each sentence as she speaks it — locally, for
free. The `emotion` message exists so a model can override that later.

## Her voice

Two engines behind one interface, chosen under **Settings → Speech**:

| Engine | Sounds like | Cost |
|---|---|---|
| Windows speech | A 2010 satnav | Installed already, starts instantly |
| Neural (Piper) | A person | ~80 MB downloaded once; 280–530 ms to first audio |

The neural engine runs **out of process** — sentences on its standard input, raw audio
on its standard output — for the same reason the local brain does: a second ONNX runtime
inside this process would sit beside Whisper's CUDA-linked one, and native dependency
collisions are not worth the milliseconds. It is downloaded on first use, into
`%APPDATA%\Octavia\voices`, from the Piper project's own GitHub release. That is an
executable rather than a model file, so it happens only when you ask for the neural
voice, and the URL is in `PiperStore.cs` where you can read it.

She starts on the Windows voice and upgrades herself when the neural engine is ready, so
a first run talks immediately instead of sitting mute through a download.

**Lip sync comes out of the audio, not the engine.** Piper reports no phoneme timings,
and neither will most of what might replace it — so the mouth is read from the sound she
is actually making: loudness for the jaw, and the balance of energy across three
formant-ish bands for the lips, measured as each buffer reaches the sound card so the
mouth is in step with what you *hear*. It is an approximation and makes no claim to know
which vowel she said. It does work for any engine, and Stage 7 needs the same DSP for
music.

To look at what it makes of a piece of speech:

```bash
dotnet run --project tools/EarsTest -- mouth C:\path\to\speech.wav
```

## The room

The wall behind her is a shader, not a colour. It runs a full day: the wall's temperature
and the key, rim and ambient lights move together through dawn, morning, evening and
night, so a warm evening key never lands on a midday wall. Two slabs drift behind her for
parallax, and a halo answers the microphone, so the room reacts before she does.

**Settings → Room lighting** pins it to an hour, or leaves it following the clock.

## Music

She hears what the machine is playing, through WASAPI loopback on the output endpoint —
no cable, no virtual device, nothing routed through her. A spectral-flux onset envelope
is autocorrelated for a tempo and matched against a pulse train for the phase, all in the
host and all local. **No model is called, which is what makes it affordable to leave
running**, and nothing is recorded: what survives the analysis is a tempo and a loudness.

Sustained music brings her headphones down onto her head; she nods on the beat, sways
across the bar, and the room's halo answers with a ring that leaves on each beat. Stop
the music and the headphones come off.

Speech is rejected by requiring three things to agree — loud enough to be something,
continuous enough not to be talking, and periodic at a steady rate. Her own voice reaches
the loopback like anything else, so the analysis is *held* while she speaks: the tempo
already found keeps running, so she stays in step with a track she is talking over and
cannot mistake herself for music.

**Settings → Hears what you play** turns it off, which closes the device rather than
ignoring it.

The same thing from the face's console, without saving it:

```js
Face.setHour(21)   // and Face.setHour(null) to follow the real clock again
```

## Diagnostics

She has to be able to explain herself on a machine nobody here can touch. Every silent
failure so far — the microphone that opened and delivered nothing, the model that would
not load — was diagnosed from *this* PC with a debugger. Elsewhere, "it doesn't work" is
all that comes back.

**The Health panel** (button in her console, or the tray) runs a self-test and shows the
result: settings, transports, renderer, microphone signal, speech model, voice and brain.
Every failing line names what to try. It is free — the self-test never calls a paid model.

**Save diagnostics** (Health → Save, or the tray) writes one zip wherever you choose:

```
README.txt    what is inside, and what to check before sending it
report.txt    versions, audio devices, and the self-test result
config.json   her settings, with anything key-shaped removed
logs/         octavia.log and its rolled predecessors
```

**The log records what she heard and said.** The bundle says so on its front page and
tells the reader to look before sending it on. The API key is not in it — it stays
DPAPI-sealed outside the bundle and is never written to the log.

When she is too broken to show her own UI, take the bundle without her:

```bash
Octavia.exe --diagnostics C:\Users\you\Desktop\octavia.zip
```

That runs no window and no session; the machine, the settings and the logs are still
what explain why she will not start.

`octavia.log` has levels and rolls at 1 MB, keeping three predecessors, so it stays
small enough to attach. `OCTAVIA_LOG` writes it somewhere else.

## What she answers, and what she lets go

A room microphone hears the television, half a phone call, and both sides of conversations
she is not in. The **attention gate** decides what deserves an answer, and it does it
without spending anything: her name and anything within the follow-up window after she
replies are let through by a rule, fragments are dropped by a rule, and only genuinely
ambiguous lines cost a call to the small local model. **No paid model is ever used to
decide whether to use a paid model.**

It fails open — if the gate cannot be reached she answers anyway, and says so in the log.
And she never ignores anything silently: a declined line appears faintly in the transcript
with the reason, because "she ignored me" has to be a question with an answer.

Typed input is never gated. If you typed it, you meant it.

```
"Gate": "local",           // or "off"
"GateModel": "llama3.2:3b", // keep this the same as LocalModel — see below
"GateFollowUpSeconds": 25,
"WakeNames": "Octavia"
```

**Keep `GateModel` and `LocalModel` the same** when running the local brain. Two models
cannot both stay resident, so a different one is evicted and reloaded on every utterance —
measured at 24 seconds against 0.7 for a warm call. The self-test fails loudly if they
differ. Avoid reasoning models here entirely: they spend their whole token budget
deliberating a yes-or-no question and return nothing.

`dotnet run --project tools/EarsTest -- gate` scores the gate against a labelled corpus
and prints what it got wrong, with the latency.

## Eyes

Ask her something that needs eyes — *"can you see what I'm holding?"* — and she takes one
still. The face owns the camera, because the face is where the person is; the host owns
the decision to open it.

**It is off by default, and it is the only sense that is.** A microphone in a room is
expected; a camera is not. Three things must all be true before it opens, and none of them
consults a model, so the rule can be audited by reading it:

1. `"Camera": true` in config
2. the words genuinely need eyes (`Sight.WantsEyes` — a plain word list)
3. the brain has eyes at all (Claude; the local model does not)

One frame, the device released in the same breath, an unmissable marker while it is live,
and nothing stored — the still rides with the one question that asked for it and never
enters the conversation history.

## The dev panel

A fourth drawer, offered on the `dev` profile and whenever the face is served without a
host. It drives `window.Face` directly: her state, mood and weight, each viseme with an
openness, a blink, where she looks, microphone level, music with a tempo and a beat, the
headphones on their own, and the room's hour.

**`Hold the face`** stops host messages that would move her from reaching the renderer, so
a mood set by hand survives the next thing she says. One row is deliberately different —
Senses — because a microphone and a loopback are devices the face does not own, so those
buttons reach the host.

`DevPanel` in `config.json` forces it on or off. The module is imported only when the
panel is opened, so a published face never loads it.

## Testing

```bash
dotnet run --project tools/EarsTest              # full suite
dotnet run --project tools/EarsTest -- mic       # capture-device diagnostic
dotnet run --project tools/EarsTest -- music     # what she makes of what is playing
dotnet run --project tools/EarsTest -- gate      # how well the attention gate judges
```

The suite synthesizes speech, runs it through Silero VAD and Whisper, asserts that
silence transcribes to nothing, exercises the streaming `<think>` filter and markdown
flattener, checks config-profile precedence, the face protocol and the diagnostics
bundle, and probes whatever local model server is configured. The exit code is the
failure count.

The config and diagnostics checks redirect themselves with `OCTAVIA_CONFIG` and
`OCTAVIA_LOG`, so running the suite never disturbs the real settings or log.

`-- mouth <file.wav>` prints the lip-sync timeline for a piece of speech.

`-- mic` lists every capture device, marks the Windows default, and reads Windows' own
meter — it separates "no audio is arriving" from "our capture is misconfigured".

`-- music` listens to whatever is playing and prints the tempo, energy and beat count as
it goes; `-- music demo` plays a track of a known tempo through the speakers first, so
the whole chain is exercised rather than only the arithmetic in the middle of it. Both
end with how completely the audio path delivered and its **crest factor** — a value near
1.5 means something is limiting everything to full scale, and no beat can be found in
audio with no dynamics left in it. Remote Desktop's audio endpoint does exactly that.

`-- beats` runs only the beat-detection checks, which need no audio device at all.

## What is real, and what is scaffolding

**Real.** Streaming replies start speaking at the first full sentence rather than waiting
for the whole response. The jaw is driven by actual SAPI viseme events, not the
timer-based guess the browser prototype used. The recognizer is muted while she speaks so
she does not transcribe herself. The key is out of the page.

**Ears.** Whisper runs locally by default (`Recognizer: "whisper"`, model
`large-v3-turbo`, downloaded to `%APPDATA%\Octavia\models` on first listen — 1.6 GB,
once). Silero VAD gates it so silence and room noise never reach the model, and bracketed
non-speech tags are filtered. On a machine with an NVIDIA GPU the CUDA runtime is picked
up automatically; otherwise it runs on CPU (choose a smaller model there — `small.en` is
quick and still beats the Windows recognizer). Set `Recognizer: "windows"` to fall back
to the old ears. A microphone that opens but delivers pure silence is reported on her
face after ten seconds rather than failing invisibly.

**Not started.** No wake word, no camera, no system-audio capture, no Home Assistant.
See `ROADMAP.md`.

## Two things to know before leaving her listening

`ListenOnStart` with continuous recognition means every recognised phrase in the room
becomes an API call. Until there is a wake word or a local gate in front of the model,
treat always-on listening as a demo rather than a mode to live in.

The prompt-caching breakpoint on the system prompt is wired but inert: the persona is a
few hundred tokens and the minimum cacheable prefix is over a thousand. It starts paying
once the persona, memory and tool definitions grow — no code change needed then.
```
