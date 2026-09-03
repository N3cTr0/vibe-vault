---
project: Octavia
tags: [octavia, architecture]
---

# Architecture

> **Since v0.26.0 the host is its own process.** `Octavia.Core` is her, `Octavia.Server.exe`
> runs her with no window, and `Octavia.exe` is a client that contains none of her. The rule
> below did not change to allow that — **the rule is what allowed it**, and the split was
> mostly a matter of moving files. See [[A Server, And Clients]].

> **Four executables since v0.46.0**, and each extra one exists for a reason that could not be
> designed around:
>
> - **`octavia-kokoro.exe`** carries its own native `onnxruntime.dll`, and `Octavia.Core`
>   carries Microsoft's for Silero and the wake word. Two in one folder is a native collision.
> - **`Octavia.Control.exe`** is her server's tray icon and settings, and it is separate
>   because **a Windows service has no desktop** — an icon drawn from session 0 is drawn where
>   nobody can see it. See [[Her Controls]].
>
> Both are the same shape of answer: a process boundary where a namespace would not do.

## The rule

**The face is a renderer; the host is the being.**

The host owns everything that thinks, hears, speaks, or holds a secret. The face owns pixels. They speak JSON. Every change should be recognisably one or the other — a stage that wants to be both is mis-designed.

Two corollaries that have already paid for themselves:

- Every renderer change must leave `OctaviaSession` untouched.
- Every sense change must leave the face untouched.

Whisper replaced the Windows recognizer without the face noticing ([[The Ears]]). A local model joined Claude without `OctaviaSession` changing shape ([[The Brain]]). The same seam is what makes a photoreal renderer a swap rather than a rewrite later — see [[Roadmap]].

`IVoice` was the fourth, and the one that proved the rule twice over: swapping Windows speech for a neural engine changed nothing in `OctaviaSession`, *and* the new engine turned out not to provide the phoneme timings the old one did — so the lip sync moved into the audio path instead, behind the same event. See [[The Voice]].

**And the rule now holds one level down.** Inside the renderer, `face.js` owns the performance and the avatar owns the anatomy, so a plaster bust and a VRM character take the same blinks, saccades, carriage and mood without either knowing about the other — see [[The Avatar Interface]]. It is the same bet as the protocol, made twice.

## Layout

```
Octavia.Core.dll — her, and nothing that draws
├── Core/         config + profiles, DPAPI secret store, paths, logging,
│                 McpServer (the config shape)
├── Brain/        IBrain -> ClaudeBrain | LocalBrain; Conversation; Speech helpers;
│   │             Moods; Persona; Situation; AttentionGate
│   └─ Tools/     ITool, McpClient (stdio JSON-RPC), ToolRegistry (+ the risk policy)
├── Senses/       ISpeechRecognizer -> WhisperRecognizer | SystemSpeechRecognizer
│   │             AudioSamples (format decoding), FaceAudioSource (a client's mic)
│   ├─ Ears/      SileroVad, WhisperTranscriber, WhisperModelStore, WhisperCompute,
│   │             WakeWord, WakeWordStore
│   └─ Music/     MusicAnalyzer — beat detection only; no device since v0.38.0
├── Voice/        IVoice -> KokoroVoice; KokoroStore, Pacer (the clock a sound card was)
├── Rounds/       IRound -> ThreatRound; Watchman (the clock), Baseline (what is normal)
├── Audio/        Fft, VisemeReader — lip sync read out of the waveform
├── Diagnostics/  SelfTest, SystemReport, DiagnosticsBundle
├── Face/         IFaceTransport -> FaceHub -> WebSocketFaceServer
│                 StaticFiles (serves wwwroot and avatars over HTTP), RemoteKey
├── Being.cs      config + socket + hub + session. The whole of what a server is
└── wwwroot/      face.js (loop + performance), environment.js (room), bust.js,
                  vrm-avatar.js, headphones.js (the prop), camera.js, watch.js,
                  dev.js, bridge.js (protocol), face.css, index.html, lib/

Octavia.Server.exe   Program.cs — load, open the socket, wait for ctrl+c
                     Service.cs — install/start/stop, and --secret
Octavia.exe          App (tray, single instance), MainWindow (WebView2, hotkey),
                     ClientConfig, Hotkey, Native. No session, and a check says so.
Octavia.Control.exe  App (tray), SettingsWindow. Her server's controls, in the
                     user's session because a service has no desktop. No session
                     either, and the same check says so.
octavia-kokoro.exe   Her voice. sherpa-onnx in a process of its own, because it
                     carries a second native onnxruntime.dll.
```

`OctaviaSession` is the only place that knows about all of them. It is deliberately the one file with wide knowledge; everything else is narrow.

**Her page lives with the core, not with the client.** It is the *server* that serves it, so a client holds no copy — which is what keeps one renderer in one place however many faces attach, and is why upgrading her page does not mean upgrading anything else.

## The four interfaces

These are the whole design. Each was introduced *before* it had a second implementation, and each earned that when the second arrived.

| Interface | Implementations | What it isolates |
|---|---|---|
| `IBrain` | `ClaudeBrain`, `LocalBrain` | What she thinks with — *not* what it thinks about; the history is passed in, since v0.24.0 |
| `ISpeechRecognizer` | `WhisperRecognizer`, `SystemSpeechRecognizer` | What she hears with |
| `IVoice` | `KokoroVoice` — `SapiVoice` and `NeuralVoice` before it | What she speaks with |
| `IFaceTransport` | `FaceHub` over `WebSocketFaceServer` | How the face is reached |
| `ITool` | `McpClient` via `ToolRegistry` | What she can *do* — an integration is a server, not a branch |
| `IRound` | `ThreatRound` | What she checks **on her own**, on a clock — see [[Her Rounds]] |

> **`IVoice` is the only one of these whose implementation has actually been replaced.** SAPI and Piper were written at the same time, against each other, so the seam had never been asked a question it could fail; swapping Piper for Kokoro in v0.40.0 was the first real test, and it passed — two files changed, and `VisemeReader`, `Pacer`, the state machine and every face were untouched. **Count how many of an interface's implementations were written before the interface**; those do not test it, they are what it was traced around.

**`IFaceTransport` learned to address one face in v0.21.0**, and it is worth saying what did *not* change: `Send(message, to)` defaults `to` to null, and null still means everyone. The session learns that faces are **distinguishable**, not how any of them connected — so the rule at the top of this note still holds exactly as written. `FaceId` is opaque; nothing can be recovered from one about a transport.

That distinction is what made the change small enough to be safe. See [[Changelog]] 0.21.0.

**In v0.26.0 it went from merging two transports to adapting one.** `WebViewFaceTransport` — a `postMessage` channel to the page hosted inside the host's own process — was deleted with the process that answered it. `FaceHub` is kept rather than collapsed into the socket server, because *merging transports is what it is for* and the second one is already specified: [[A Server, And Clients]] item 3 has the client lending the server its devices, and that arrives here as a peer of the socket rather than a special case inside it.

> One consequence worth knowing: `BuiltInFace` is now always null, and honestly so. It meant "the renderer that is always there", and with a server there is no such thing — every face comes and goes over a socket.

**In v0.24.0 it learned to address a set of them**, and the same care applies: `SendMany` and `SendAudio` take a list of `FaceId`, never a room. The session knows which faces are in a room; the transport is told *who* and never *why*. See [[One Being, Many Rooms]].

**`IBrain` stopped owning the conversation in the same release.** It held the one `Conversation` there was, and `Forget()` cleared it — which stopped being right the moment she could be in two rooms. `RespondAsync(history, ...)` now takes it, so N conversations share one client, one key and one set of tools. This is the interface getting *smaller*, which is the direction that has worked here: a `ClaudeBrain` per room would have duplicated everything expensive in order to keep two lists of strings apart.

`ITool` is the newest and the only one still waiting for its second implementation to prove it. It follows the same discipline as the others — introduced before it was needed — and the seam, the stdio client, the risk policy and the confirmation rule for irreversible actions are all built and tested. What is missing is the brain-side loop that calls one. See [[Roadmap]] stage 12.

And one more, in the renderer rather than the host: the **avatar** — the bust and a VRM take the same performance. See [[The Avatar Interface]].

## The message protocol

Host and face exchange JSON over a loopback WebSocket, with postMessage as the fallback when the port cannot bind. Full reference in the repo's `PROTOCOL.md`, mirrored at [[Face Protocol]]; the reasoning is in [[The Host-Face Bridge]].

**The message lists are deliberately not repeated here.** [[Face Protocol]] is the mirror of `PROTOCOL.md` and is regenerated from it on every vault sync, so it cannot drift; a hand-kept copy in this note can, and did — it sat five face→host messages and eight `hello` fields behind the code until v0.20.1.

That stopped being a documentation problem the moment a second face existed. An Android client is being written against that contract by someone who cannot read `OctaviaSession`, and the gap was found the hard way: `say` carries `text` while every `set*` message carries `value`, the first implementation guessed `value`, and she accepted it and silently did nothing — no error, no log line, a dead button. **A contract that is quietly incomplete is worse than one that is honestly small.** Read the mirror, not this note.

The face holds no key, makes no model calls and owns no audio. Its CSP is `default-src 'none'`; `connect-src` names the loopback socket, the read-only avatar origin the host maps, and `blob:` — so even the character file it renders is chosen for it.

> `blob:` is on that list for a real reason. glTF carries its textures inside the binary, three.js decodes them to a `Blob` and loads them from a `blob:` URL, and its absence silently blocked **every texture in every model** for the project's entire life. She had never once rendered with a face. See [[Changelog]] 0.12.0.

## A turn, end to end

1. **Hearing.** Microphone → `MicLevelMeter` (amplitude → `level`, so the face reacts while you are still talking) and `WhisperRecognizer` in parallel. Silero VAD marks utterance boundaries; only audio it vouches for reaches Whisper.
2. **Thinking.** `OctaviaSession.RespondTo` mutes the ears (so she never transcribes herself), sets `thinking`, and streams from the brain.
3. **Speaking.** The brain yields **one sentence at a time**, so the voice starts speaking before the model has finished writing. This is the single biggest contributor to her feeling responsive. Each sentence is also read for a mood, locally and for free, and an `emotion` goes to the face when it changes — see [[The Voice]].
4. **Moving.** Where the visemes come from depends on the engine, and the face cannot tell: Windows speech raises real phoneme events, while the neural engine's mouth is read out of its own audio as it reaches the sound card. Either way the face receives openness plus a VRM mouth shape.
5. **Settling.** When the stream ends *and* the speech queue drains, the ears unmute and she returns to listening or idle.

The ordering in step 5 matters: sentences arrive over time, so the speech queue can empty mid-reply. `_responding` guards against settling early.

## Why the face is served over https

`SetVirtualHostNameToFolderMapping` maps `wwwroot` to `https://octavia.face`, rather than loading `file://`. That makes the page a **secure context**, which the camera and microphone permissions in later stages depend on, and it avoids `file://` origin restrictions. Free to do now, expensive to retrofit.

## And why that was not enough *(v0.20.0)*

The mapping above is a **WebView2 feature, not a server.** Nothing in this process had ever answered a GET, which meant the "transport that can leave the machine" recorded in v0.14.0 was half of one: a phone could open the socket and still have no page to run in it.

`StaticFiles` + `WebSocketFaceServer` close that. The listener was already parsing request lines for the WebSocket handshake, so answering a GET is a branch rather than a second server — and the page and the socket then share one origin and one port.

**The deciding fact was not preference but the avatars.** Vendoring `wwwroot` into a client was the alternative, and it lost because there are *two* virtual host mappings: the second serves her avatars folder, and a `.vrm` is user data — git-ignored, chosen at runtime, in no repository — so it can never be baked into a client. The host had to serve files regardless. The real choice was one mechanism or two of them drifting.

Sub-resources are the subtlety: `<link href="face.css">` and `import('./watch.js')` are resolved by the browser, which knows nothing about a key, so gating them on `?key=` would have served the page and refused everything in it. The credential is echoed back as an `HttpOnly`, `SameSite=Strict` cookie and the assets present that instead.

## What deliberately stays out-of-process

The local model runs in Ollama (or any OpenAI-compatible server), not in-process. An in-process runtime would put a second CUDA-linked native stack alongside `Whisper.net.Runtime.Cuda`, and later Audio2Face. One process should not host two of them. See [[Choosing a Local Model]].
