---
project: Octavia
tags: [octavia, architecture]
---

# Architecture

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
Octavia.exe (WPF)
├── Core/         config + profiles, DPAPI secret store, paths, logging, hotkey,
│                 McpServer (the config shape), Forget()
├── Brain/        IBrain -> ClaudeBrain | LocalBrain; Conversation; Speech helpers;
│   │             Moods; Persona; Situation; AttentionGate
│   └─ Tools/     ITool, McpClient (stdio JSON-RPC), ToolRegistry (+ the risk policy)
├── Senses/       MicLevelMeter; ISpeechRecognizer -> WhisperRecognizer | SystemSpeechRecognizer
│   │             AudioSamples (format decoding), AudioDevices (resolution by name)
│   ├─ Ears/      SileroVad, WhisperTranscriber, WhisperModelStore, WhisperCompute
│   └─ Music/     LoopbackListener, MusicAnalyzer, MusicWatcher
├── Voice/        IVoice -> SapiVoice | NeuralVoice; PiperStore
├── Audio/        Fft, VisemeReader — lip sync read out of the waveform
├── Diagnostics/  SelfTest, SystemReport, DiagnosticsBundle
├── Face/         IFaceTransport -> FaceHub -> WebViewFaceTransport + WebSocketFaceServer
│                 StaticFiles (serves wwwroot and avatars over HTTP), RemoteKey
└── wwwroot/      face.js (loop + performance), environment.js (room), bust.js,
                  vrm-avatar.js, headphones.js (the prop), camera.js, watch.js,
                  dev.js, bridge.js (protocol), face.css, index.html, lib/
```

`OctaviaSession` is the only place that knows about all of them. It is deliberately the one file with wide knowledge; everything else is narrow.

## The four interfaces

These are the whole design. Each was introduced *before* it had a second implementation, and each earned that when the second arrived.

| Interface | Implementations | What it isolates |
|---|---|---|
| `IBrain` | `ClaudeBrain`, `LocalBrain` | What she thinks with |
| `ISpeechRecognizer` | `WhisperRecognizer`, `SystemSpeechRecognizer` | What she hears with |
| `IVoice` | `SapiVoice`, `NeuralVoice` | What she speaks with |
| `IFaceTransport` | `FaceHub` over `WebViewFaceTransport` + `WebSocketFaceServer` | How the face is reached |
| `ITool` | `McpClient` via `ToolRegistry` | What she can *do* — an integration is a server, not a branch |

**`IFaceTransport` learned to address one face in v0.21.0**, and it is worth saying what did *not* change: `Send(message, to)` defaults `to` to null, and null still means everyone. The session learns that faces are **distinguishable**, not how any of them connected — so the rule at the top of this note still holds exactly as written. `FaceId` is opaque; nothing can be recovered from one about a transport.

That distinction is what made the change small enough to be safe. See [[Changelog]] 0.21.0.

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
