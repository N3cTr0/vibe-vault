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
├── Core/         config + profiles, DPAPI secret store, paths, logging, hotkey, Forget()
├── Brain/        IBrain -> ClaudeBrain | LocalBrain; Conversation; Speech helpers; Moods
├── Senses/       MicLevelMeter; ISpeechRecognizer -> WhisperRecognizer | SystemSpeechRecognizer
│   ├─ Ears/      SileroVad, WhisperTranscriber, WhisperModelStore
│   └─ Music/     LoopbackListener, MusicAnalyzer, MusicWatcher
├── Voice/        IVoice -> SapiVoice | NeuralVoice; PiperStore
├── Audio/        Fft, VisemeReader — lip sync read out of the waveform
├── Diagnostics/  SelfTest, SystemReport, DiagnosticsBundle
├── Face/         IFaceTransport -> FaceHub -> WebViewFaceTransport + WebSocketFaceServer
└── wwwroot/      face.js (loop + performance), environment.js (room), bust.js,
                  vrm-avatar.js, headphones.js (the prop), bridge.js (protocol),
                  face.css, index.html, lib/
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

And a fifth, in the renderer rather than the host: the **avatar** — the bust and a VRM take the same performance. See [[The Avatar Interface]].

## The message protocol

Host and face exchange JSON over a loopback WebSocket, with postMessage as the fallback when the port cannot bind. Full reference in the repo's `PROTOCOL.md`, mirrored at [[Face Protocol]]; the reasoning is in [[The Host-Face Bridge]].

**Host → face:** `hello`, `state`, `level`, `viseme`, `emotion`, `music`, `caption`, `turn`, `notice`, `needKey`, `cleared`, `diagnostics`, `diagnosticsSaved`.

**Face → host:** `ready`, `say`, `listen`, `hush`, `forget`, `setKey`, `setVoice`, `setVoiceEngine`, `setAvatar`, `setRoomHour`, `setMusic`, `selfTest`, `saveDiagnostics`, `openDataFolder`, `faceError`.

The face holds no key, makes no model calls and owns no audio. Its CSP is `default-src 'none'`, and `connect-src` names only the loopback socket and the read-only avatar origin the host maps — so even the character file it renders is chosen for it.

## A turn, end to end

1. **Hearing.** Microphone → `MicLevelMeter` (amplitude → `level`, so the face reacts while you are still talking) and `WhisperRecognizer` in parallel. Silero VAD marks utterance boundaries; only audio it vouches for reaches Whisper.
2. **Thinking.** `OctaviaSession.RespondTo` mutes the ears (so she never transcribes herself), sets `thinking`, and streams from the brain.
3. **Speaking.** The brain yields **one sentence at a time**, so the voice starts speaking before the model has finished writing. This is the single biggest contributor to her feeling responsive. Each sentence is also read for a mood, locally and for free, and an `emotion` goes to the face when it changes — see [[The Voice]].
4. **Moving.** Where the visemes come from depends on the engine, and the face cannot tell: Windows speech raises real phoneme events, while the neural engine's mouth is read out of its own audio as it reaches the sound card. Either way the face receives openness plus a VRM mouth shape.
5. **Settling.** When the stream ends *and* the speech queue drains, the ears unmute and she returns to listening or idle.

The ordering in step 5 matters: sentences arrive over time, so the speech queue can empty mid-reply. `_responding` guards against settling early.

## Why the face is served over https

`SetVirtualHostNameToFolderMapping` maps `wwwroot` to `https://octavia.face`, rather than loading `file://`. That makes the page a **secure context**, which the camera and microphone permissions in later stages depend on, and it avoids `file://` origin restrictions. Free to do now, expensive to retrofit.

## What deliberately stays out-of-process

The local model runs in Ollama (or any OpenAI-compatible server), not in-process. An in-process runtime would put a second CUDA-linked native stack alongside `Whisper.net.Runtime.Cuda`, and later Audio2Face. One process should not host two of them. See [[Choosing a Local Model]].
