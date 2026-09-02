---
project: Octavia
tags: [octavia, code]
source-path: ROADMAP.md
---

# ROADMAP.md

```markdown
# Octavia — Roadmap

Target: an always-on companion with a fully animated, eventually photoreal 3D face that
listens locally, reacts to music (headphones on, dancing), has a living background, and
later gets eyes and hands. This file is the order of work and the reasoning; strike
through stages as they land.

The architecture rule that makes all of it possible: **the face is a renderer, the host
is the being.** Every stage below is either a better renderer or a better host — never
both at once.

---

## ~~Stage 1 — Ears: VAD + Whisper~~ *(done 08/29/2026)*

The biggest quality jump available, and the foundation for everything acoustic.

- **Silero VAD** (ONNX, tiny, CPU) decides when speech starts and ends. It gates
  everything: without it Whisper hallucinates text out of silence ("Thank you." is the
  classic), and with it we stop transcribing the television.
- **Whisper** via `Whisper.net` + the CUDA runtime package, as a second
  `ISpeechRecognizer` — the interface already exists, nothing else changes.
- Model file is a config choice, downloaded on first run:
  - `large-v3-turbo` — the **default for conversation**. ~1–2% worse WER than large-v3,
    several times faster, and latency is what makes a spoken exchange feel alive.
  - `large-v3` — "the biggest one," a config flip away when accuracy beats snappiness.
  - The deciding factor on the target PC is not whether the GPU *can* run large-v3 —
    it easily can — but that the same GPU will later render the face and run
    Audio2Face. Turbo leaves headroom; large-v3 is there when wanted.
- Confidence gate carries over (`MinConfidence` maps to Whisper's avg-logprob).

**Exit test:** mumble at her from across the room over music; she gets the words.

## ~~Stage 2 — A local brain, and dev profiles~~ *(done 08/29/2026)*

Free, offline replies so every later stage can be tested by talking to her for an hour
without thinking about the bill. Not throwaway scaffolding — see "why it stays" below.

- **`IBrain`** interface; today's `Brain` becomes `ClaudeBrain`, joined by
  `LocalBrain`. `OctaviaSession` keeps its shape, exactly as `ISpeechRecognizer` let
  Whisper in without disturbing anything.
- **`LocalBrain` speaks the OpenAI-compatible chat API over plain HTTP.** That one
  decision means it works with Ollama, LM Studio, `llama-server`, or anything else
  that speaks the same shape — model swaps become a config edit, never a rebuild.
- **Deliberately out-of-process.** An in-process runtime (LLamaSharp) would put a
  second CUDA-linked native stack in the same process as `Whisper.net.Runtime.Cuda`,
  and later Audio2Face. A separate server sidesteps native-dependency conflicts
  entirely and can load and unload models without restarting her.
- Model: a 3–4B instruct model is plenty to exercise the pipeline — Qwen3 4B or
  Gemma 3 4B, ~3 GB at Q4.
- **Dev profiles.** The dev VM and the eventual GPU box want opposite settings.
  Introduce named config profiles so one flag switches the whole rig:
  - *dev* — `small.en` Whisper, `LocalBrain`, no spend, runs on a laptop or VM
  - *live* — `large-v3-turbo`, `ClaudeBrain`, the real thing

**Why it stays after dev.** Stage 9 needs a cheap local gate in front of the expensive
model — something to judge "was that addressed to me, and is it worth answering?"
before spending a Claude call. That gate is this same local model and this same
interface. Building it now means always-on listening later is a config change rather
than a new subsystem.

**Honest caveat:** a 4B model will *not* hold the persona — expect markdown, rambling,
and stage directions no matter what the system prompt says. It is a test of the
*pipeline*, not of her character. Judge lip sync, latency and turn-taking with it;
judge Octavia herself on Claude.

**Exit test:** unplug the network, talk to her for ten minutes, watch the face work.

## ~~Stage 3 — Cut the cord: WebSocket protocol~~ *(done 08/30/2026)*

Small stage, huge unlock. Replace the WebView2 postMessage transport with a local
WebSocket server in the host (`IFaceTransport` was built for exactly this swap), and
write down the message schema in `PROTOCOL.md` with a version field.

After this, *anything* that speaks the protocol is a legal face: the WebView2 page, a
browser on a wall tablet, and — critically — the Unreal renderer in Stage 8. This is
what makes the photoreal stage a swap instead of a rewrite.

## ~~Stage 4 — Diagnostics: make her debuggable in someone else's hands~~ *(done 08/30/2026)*

Before she goes anywhere near another machine, she has to be able to explain herself.
Every failure so far — the silent microphone, the invisible `<think>` truncation, the
model that would not load — was diagnosed from *this* machine with a debugger and a test
harness. On someone else's PC there is none of that, and "it doesn't work" is all that
comes back.

- **Structured logging.** `octavia.log` grows forever, has no levels and no rotation.
  Give it levels, a size cap with roll-over, and one line per meaningful event
  (transport attached, model loaded, turn started, error). Errors get a stack trace;
  everything else stays readable.
- **Real crash handling.** `DispatcherUnhandledException` currently swallows and
  continues, which hides exactly the faults worth seeing. Log it properly, tell the
  user something broke, and offer the bundle below.
- **"Save diagnostics" — the button that matters.** A file dialog asking where to put a
  single zip, so a non-technical user can email it back. It should contain more than the
  log:
  - the log (current + rolled)
  - `config.json`, **with anything sensitive stripped**
  - app version, .NET version, Windows build
  - audio device inventory and whether the default input carries signal — the RDP failure
    would have been obvious from this alone
  - which Whisper model is present, whether the local brain endpoint answers, whether the
    face socket bound
  - recent `faceError` messages from the renderer
- **Privacy, deliberately.** The log will contain transcripts of things said in the room.
  The bundle must say plainly what it holds, never include the API key (it never leaves
  DPAPI anyway), and ideally let the user open the folder and look before sending. Get
  this right now — it is much harder to retrofit trust.
- **A diagnostics panel in the face.** One screen answering "is she working?": transports
  attached, brain reachable, ears engine and model, microphone device and live level,
  versions. Most support questions die here.
- **A self-test she can run herself.** The checks `tools\EarsTest` already performs, but
  in-app and on demand: is the mic carrying signal, does the brain answer, is the model
  present, did the socket bind. Turn each red line into a sentence naming the fix.

The through-line: every silent failure found so far becomes a visible one. The silence
watchdog from Stage 1 was the first instance of this; this stage generalises it.

**Exit test:** hand the published exe to someone on another machine, have them break it,
and diagnose it from the file they send back without touching their PC.

**What it caught on the way in.** Building the stage found two faults of exactly the kind
it exists to expose, both silent: a file dialog constructed off the UI thread threw into a
discarded task and did nothing at all, and the headless bundle blocked the dispatcher on
its own continuations and hung forever. Every fire-and-forget now logs its failure.

## ~~Stage 5 — The real face: VRM avatar + dynamic background~~ *(done 08/30/2026)*

Retire the plaster bust. The face page loads a **VRM avatar** (`@pixiv/three-vrm` on
three.js) — an actual character: skin, hair, eyes, and the standard **52 ARKit-style
blendshapes** driven from the protocol instead of one jaw morph.

- Viseme events map to mouth blendshapes (the SAPI viseme→openness table generalises
  to viseme→blendshape-set).
- Expression states: idle / listening / thinking / speaking get real faces, plus an
  `emotion` message so the brain can tag replies (smile, concern, surprise).
- **Props live here too: the headphones.** A stylized 3D rig is what makes "she puts
  on headphones" *feasible* — it's an attach-bone and a short animation, where a
  photoreal neural face could never hold a prop.
- **Dynamic background:** the flat gallery wall becomes a shader scene — time-of-day
  lighting, slow parallax depth, and an audio-reactive mode fed by the same `level`
  and (later) `music` messages. Cheap on GPU, transforms the presence.

Avatar sourcing: VRoid Studio (free) to design her, or a commissioned VRM. The model
is a file the face loads — she can have wardrobe.

**As built.** three.js went from the 2021 UMD build to r180 as ES modules first, because
`@pixiv/three-vrm` needs r158+ and everything else would have been written twice. The
renderer split into an environment, an avatar and a face that drives whichever avatar it
has, through one small interface: `setViseme`, `setExpression`, `setGaze`, `setBlink`,
`setPose`, `update`. The bust implements it and so does a VRM, which is the same bet as
Stage 3's protocol one level down.

**The decision that paid for itself:** the expression vocabulary *is* VRM 1.0's —
`happy / angry / sad / relaxed / surprised / neutral` and the visemes `aa / ih / ou /
ee / oh`. So the protocol maps to a real character by identity, with no translation layer
to get wrong. Verified against a real VRM: every name we send exists on the model.

**Still open:** headphones as an attached prop wait for Stage 7, where there is music to
put them on for. Octavia has no character file of her own yet — that is a design
decision, not an engineering one.

## ~~Stage 6 — A voice worth the face~~ *(done 08/30/2026)*

SAPI voices will break the illusion the moment the face stops being a statue. Swap in
a **local neural TTS** (Kokoro or Piper class, GPU or CPU) behind the same `VoiceBox`
event surface: audio out plus phoneme/word timings driving the blendshapes. The
sentence-streaming pipeline already feeds it. Voice becomes a character decision, not
a Windows setting.

**As built.** `IVoice` joins `IBrain` and `ISpeechRecognizer`: `SapiVoice` is what she
had, `NeuralVoice` runs **Piper** as a long-lived child process — the same
out-of-process reasoning as the local brain, so a second ONNX runtime never sits beside
Whisper's. Sentences go in on stdin, raw PCM comes back on stdout and is played through
NAudio. The engine and the voice are fetched on first use, like the Whisper models.

**The part the plan got wrong.** It assumed the engine would hand us phoneme timings.
Piper hands over a waveform and nothing else — and so, most likely, will whatever
replaces it. So the mouth is now **read out of the audio itself**: loudness for the jaw,
and the balance of energy across three formant-ish bands for the lips, analysed at the
moment each buffer reaches the sound card so the mouth is in step with what is *heard*
rather than what has been generated. It is an approximation and does not claim to know
which vowel she said, but it works for any engine we ever swap in — and Stage 7 needs
exactly this DSP for music.

Measured on the dev VM (3 cores, no GPU): **first audio 280–530 ms** after a sentence is
handed over, and synthesis of the next sentence overlaps playback of the current one.
Windows speech stays as the instant, nothing-to-download fallback.

## ~~Stage 7 — Music: headphones on, dance~~ *(done 08/30/2026)*

- **WASAPI loopback** (NAudio) hears what the PC is playing — this was the original
  reason the app couldn't stay a browser page.
- Local DSP in the host — onset/beat detection, BPM, energy, a music-vs-speech
  check — emits `music { bpm, energy, beat }` over the protocol. **No model calls;
  dancing is free.**
- The face responds: headphones animation on sustained music, head-bob and sway
  locked to the beat, background pulsing with energy. Stop the music, headphones
  come off.
- The same signal becomes context for the brain ("what's playing?" comes later with
  fingerprinting; "she knows there *is* music" comes now).

**As built.** All of it, and the DSP the previous stage predicted it would want: a
spectral-flux onset envelope, autocorrelated for a tempo, matched against a pulse train
for the phase. `MusicAnalyzer` takes samples and returns numbers — no device, no
threads — so its tempi are checked against generated tracks rather than by playing
something and watching her.

**The two decisions worth keeping.** Her own voice comes back through the loopback like
anything else, so the analysis is *held* while she speaks rather than stopped: the tempo
already found keeps running, she stays in step with a track she is talking over, and she
cannot hear herself and call it music. And the brain is told about the music on the
current question only — not in the system prompt, which would void the cache breakpoint,
and not in the history, which would leave it insisting there is music an hour later.

**What the plan got wrong.** It assumed hearing the machine was the hard part. Hearing it
took an afternoon; *believing* what came back took longer. The tempo would not settle on
this VM, and the cause was neither the arithmetic nor the capture — Remote Desktop's
audio endpoint normalises everything to full scale, and a beat cannot be found in audio
with no dynamics left in it. The analyser is verified offline at every tempo and both
common sample rates; the live path is verified only as far as this machine allows, and
`EarsTest -- music` now measures the crest factor so the next person is told rather than
left guessing.

## Stage 8 — Photoreal: the decision gate *(decided 08/30/2026; rendering waits on the GPU box)*

> **Audio2Face is not load-bearing — researched 08/31/2026, when a new PC with an AMD card
> was proposed.** [Audio2Face-3D needs CUDA and TensorRT](https://docs.nvidia.com/ace/audio2face-3d-microservice/latest/text/support-matrix.html)
> on an Ampere, Ada or Blackwell card; open-sourcing it under MIT changed the licence, not
> the dependencies. That reads like an NVIDIA lock-in for this stage. **It is not**, and
> the reason is the protocol.
>
> Audio-to-blendshape is a populated category now, and every serious option emits **ARKit
> blendshapes** — which is already the vocabulary here, because Stage 5 deliberately took
> VRM 1.0's. Anything in this list drops into the same seam:
>
> | Option | Emits | Vendor | Note |
> |---|---|---|---|
> | [Epic's Audio-Driven Animation](https://dev.epicgames.com/documentation/metahuman/audio-driven-animation) | MetaHuman rig directly | **neutral** | In MetaHuman Animator since UE 5.6. Epic's own hardware guidance lists an RX 5500 XT. Nothing extra to install. **⚠ Offline — see the correction below.** |
> | [NeuroSync](https://neurosync.info/) | 61 ARKit + 7 emotion | neutral | Transformer, streams to UE5 over LiveLink. **CC BY-NC 4.0** — fine personally, not commercially. |
> | [fotonlabs/unreal-audio2lipsync](https://huggingface.co/fotonlabs/unreal-audio2lipsync) | 52 ARKit @ 60 fps | neutral | Straight into the MetaHuman face rig. |
> | Audio2Face-3D | ARKit | **NVIDIA only** | Best-known, and the one that costs you the choice. |
>
> **The first option is the answer if she is going to be a MetaHuman anyway**: Epic's own
> audio-driven animation is built into the tool that renders her, vendor-neutral, and needs
> no second inference stack sharing the card.
>
> ### Correction, 09/02/2026 — that option is *offline*, and the note above was wrong
>
> **Audio-Driven Animation bakes an Animation Sequence from a `SoundWave` asset.** It is an
> editor workflow, not a runtime one, and Epic's own documentation draws the line explicitly:
> even setting a Performance asset to the *Realtime Audio Solver* "is still an offline
> process and is not the same as generating animation in real time". Octavia needs the live
> path — sentences arriving as PCM while she speaks — so this was picked for the wrong job.
>
> **The live path exists and is a different feature**: the **MetaHuman Audio Live Link
> Source** (UE 5.6+, MetaHuman Live Link plugin). It got materially better in **MetaHuman
> 5.8**, which shipped a new real-time audio model with procedural blinks and *automatic
> emotion detection from the audio* — worth noting, because `emotion` is currently read from
> her text locally, and a solver inferring it from the sound is a second opinion on the same
> question.
>
> **The catch is architectural, not technical.** The Audio Live Link Source consumes a
> **Windows audio capture device**. Her voice reaches a face as **PCM over a WebSocket**.
> Those do not meet, and no amount of GPU fixes it. Three bridges, cheapest first:
>
> 1. **Ship the `viseme` message.** She has computed real-time visemes off `MouthTap` since
>    Stage 6 — openness plus one of five shapes, ~40 Hz, in step with what you hear, already
>    crossing the protocol and already proven by two renderers. **A MetaHuman can consume
>    that on day one.** It is coarser than 52 ARKit curves and it costs nothing, adds no
>    dependency, and needs no new hardware. Then the animation source is upgraded behind an
>    unchanged protocol — which is the exact swap this stage claims to be about.
> 2. **A runtime lip-sync plugin** that takes streamed buffers from any source (the FAB
>    *Runtime MetaHuman Lip Sync* is the obvious one) — PCM in, ARKit out, in-process.
> 3. **A virtual audio cable**, so the Live Link Source sees a device. Zero code, and it
>    pins the renderer to whichever machine is playing her voice — which the device rule
>    below now forbids anyway.
>
> **The lesson is the one already in the vault**, arriving a third time: a plausible option
> was written down as a finding after a single reading, and it stood for two days shaping a
> hardware decision. What falsifies it is one sentence of Epic's own docs.
>
> ### AMD, and one concrete report
>
> Epic's recommended spec lists AMD explicitly (RX 6800 XT, 8 GB VRAM), MetaHuman Animator
> needs DX12, and AMD supports it — so nothing here contradicts the RX 9070 XT plan. But
> there is now a real report of **UE 5.8 + MetaHuman Animator on an RX 6600 crashing with
> "GPU Crashed or D3D Device Removed"**, diagnosed as a TDR timeout and worked around with
> registry TDR delays, disabling HAGS, or forcing DX11. One report, RDNA2, unconfirmed on
> RDNA4 — not a reason to change the card, and the first concrete AMD-specific MetaHuman
> problem worth knowing about before spending the money.
>
> So the card choice is about *rendering* and about local inference, not about whether this
> stage is possible. AMD notes: [`Whisper.net.Runtime.Vulkan`](https://www.nuget.org/packages/Whisper.net.Runtime.Vulkan)
> exists, so Whisper is GPU-accelerated on AMD; Ollama is messier on Windows, where the
> official installer still ships ROCm 6.4.2 and RDNA4 needs ROCm 7.x — so it means
> [experimental Vulkan](https://docs.ollama.com/gpu) or a community fork until upstream
> catches up.
>
> **Decided 08/31/2026: the AMD build.** Radeon RX 9070 XT (or its successor if one lands
> first), Ryzen 9 9950X3D, 128 GB DDR5. **Stage 8 therefore plans on Epic's Audio-Driven
> Animation rather than Audio2Face**, which is the better answer regardless — it lives
> inside the renderer instead of competing with it for the card.
>
> Two things that follow, and are easy to forget later:
>
> - **Nothing in this repo should assume CUDA.** `Whisper.net.Runtime.Cuda` is still
>   referenced in the `.csproj` and will simply never load on that machine; the
>   `WhisperCompute` setting added in v0.12.0 already handles choosing, but `gpu` will mean
>   Vulkan there rather than CUDA, and the runtime package needs adding before it can.
> - **Re-measure everything on arrival.** Every figure in this repo and the vault —
>   `EarsTest models`, `EarsTest compute`, the gate median, the music crest — was taken on
>   an 8-core CPU with a GT 730 doing nothing useful. The 128 GB is the part that changes
>   the design rather than just the numbers: a small fast gate and a genuinely large brain
>   can both stay resident, which is the split v0.15.0 established at 3B and 7B.


The reference image — a photoreal woman — is a different rendering class from Stage 5.
Two credible routes, decided *when we get here* by how each has matured:

1. **MetaHuman.** Epic's 2025 license change made MetaHumans usable in any engine and
   free under $1M revenue. Practical path: an Unreal app as the face, attached over
   the Stage-3 WebSocket, same protocol. Full photoreal skin/hair/eyes, and props
   (headphones!) are ordinary animation work.
2. **Audio2Face-3D.** NVIDIA open-sourced it (MIT): real-time audio→blendshape
   inference on the GPU. It replaces the *animation source* (better than viseme
   mapping, with emotion inference) and feeds either the VRM face or a MetaHuman.

Likely answer is **both**: Audio2Face for animation, MetaHuman/Unreal for rendering,
Octavia.exe unchanged as the being. This is where the latest-GPU PC earns its keep —
budget it as renderer + Whisper + Audio2Face + local gate sharing one card.

### The decision, 08/30/2026

**Both, as predicted — but the reasoning has changed, and so has the order.**

*MetaHuman for rendering.* Not because it is the most photoreal option; it is not. The
deciding requirement is that **she wears headphones**, which arrived in Stage 7 and now
works. A rigged character can hold a prop, turn its head and be *directed*. The neural
alternatives cannot, and no amount of realism substitutes for that here — an assistant
that cannot put headphones on is a worse Octavia than a slightly less real one.

*Audio2Face-3D for animation.* It is MIT, its SDK runs local inference in C++/CUDA at
better than 60 fps, and it emits **ARKit blendshapes** — which is what a MetaHuman speaks
natively. It needs CUDA 12.8+ and TensorRT 10.13+ on a 4 GB+ card.

*Gaussian splatting, the option this list did not have.* It is now the most photoreal
real-time head available and it is genuinely better at looking like a person. It is
rejected anyway: research-grade, no prop handling, and nothing to direct with an
expression name. Worth revisiting when it can be told to look left.

**Where Audio2Face runs: the host, out of process.** NVIDIA ships an Unreal plugin, so
putting it in the renderer is the obvious choice — and it is the wrong one. The host owns
the voice and already has the PCM in hand at the `MouthTap` that Stage 6 put between the
buffer and the sound card. Running it there means the audio never has to cross the
protocol, every face benefits including the VRM, and Octavia.exe keeps its shape. It goes
out of process for the standing reason: it would otherwise be a second CUDA runtime beside
Whisper's. That is the Piper arrangement exactly, and Piper works.

**A dated constraint, discovered on the way.** The MetaHuman Creator *web app* shuts down
**11/05/2026**; creation moves into Unreal Engine 5.7's in-editor tooling. That is roughly
nine weeks from this decision. It does not change the choice, but it does mean any
character made the easy way has to be made and exported before then.

**What was actually delivered here.** The stage's premise — "photorealism becomes
attaching a different renderer" — was an assumption. It is now a tested claim:
`tools\attach-face.ps1 -Conformance` drives a running host through a turn and checks every
message a renderer must receive, and `PROTOCOL.md` states what a renderer must implement.
It found a real gap on its first run: a face attaching mid-session was never told her
current expression, because `emotion` is only sent on change. `hello` now carries her
state and her mood.

**What is blocked, and honestly so.** The rendering itself. This VM has a VMware SVGA 3D
adapter — no CUDA, three cores — and both routes need a real NVIDIA card. Unreal, the
MetaHuman and Audio2Face all wait for that machine. Nothing else in this stage does.

## Stage 9 — Eyes and hands *(gate done 08/30/2026; eyes built; a camera arrived 09/01/2026, still unopened)*

- Camera frames to the brain on demand (Sonnet is multimodal; "can you see me?" works).
- Presence detection locally so she notices you arrive.
- **Home Assistant** as tool definitions — deliberately deferred until she's worth
  talking to. The Brain grows a tool loop; the skill is knowing which requests need
  tools at all.
- Wake word (openWakeWord), plus the local-model gate from Stage 2 deciding what is
  worth a Claude call. This is what makes always-on listening affordable.

### As built, 08/30/2026 — the gate and the eyes

**The gate is the stage.** It is what Stage 2 was really building when it added a local
model, and it works: a room microphone hears the television, half a phone call, and both
sides of conversations she is not in, and none of that now reaches a paid model.

Two layers, cheapest first. **Rules settle the clear cases for nothing**: her name is
always let through, anything within the follow-up window after she answers is let through
(without which her name would have to be said every single turn), and fragments are
dropped. Only genuinely ambiguous lines cost a model call, and that call goes to the small
local model, which is free. *No paid model is ever used to decide whether to use a paid
model.*

It **fails open**. A companion who goes silent because a helper model stopped answering is
broken; one who occasionally replies to the television is merely annoying. And she never
ignores anything *silently* — a rejected line goes to the log and to the face as
`overheard`, with the reason, because "she ignored me" has to be a question with an answer.

Measured on this VM, 18 labelled lines: **14 agreed, 1 ignored-you, 3 answered-noise,
median 1.2 s.** The error profile matters more than the score — a false no is the one that
makes her feel broken.

**Two things the plan had wrong.** The gate model should be the *same* model as the brain,
not a smaller one: a separate model is evicted and reloaded on every utterance, measured
at 24 seconds against 0.7 for a warm call. And a reasoning model is useless as a gate — it
spends its whole token budget deliberating a yes-or-no question and returns nothing. There
is no portable way to switch that off; `think`, `/no_think` and `chat_template_kwargs` were
each tried and each ignored.

**The eyes are built, and honestly half-verified.** The face owns the camera — that is what
the virtual `https` origin was set up for in Stage 3 — and the host owns the decision. One
still, only when the words genuinely need eyes, device released in the same breath, and an
unmissable marker while it is live. The camera is the **only sense that is off by default**:
a microphone in a room is expected, a camera is not.

Since verified against a real redirected webcam: the host grants the permission, the frame comes back 768x432 with genuine detail, and the whole path works. Three things only hardware could have found - see v0.9.1. Everything up to the glass is tested too: the intent rules, the no-camera path, the refusal
path, the marker not sticking. **A frame has now been captured here**, once a webcam was
redirected in. What is still unproven is only the last step: no still has yet been sent to
Claude, because the dev profile's brain has no eyes and that call costs money.

**Wake word: not built, and arguably not wanted.** openWakeWord ships models for "hey
jarvis" and the like; there is no "Octavia", and training one needs a corpus that does not
exist. Meanwhile the free layer already does the cheap half of the job — her name is
matched in the transcript for nothing. What a real wake word would buy is not running
Whisper at all, which is a battery argument on a machine that is plugged in.

**Presence detection is not built.** It needs the camera, and it waits with it.

> **A camera arrived 09/01/2026** — a spare USB one, plugged into the host. It enumerates
> as a generic UVC `USB Video Device`, status OK, with a microphone of its own. So "the
> camera cannot be tested here" is no longer true, and this is unblocked whenever it is
> worth doing.
>
> ~~**It has not been opened.** `Camera` is `false` in config and should stay there until
> somebody chooses otherwise; what is verified is that the device exists, not that a frame
> comes back. Two things still gate a real end-to-end test: `MaybeLookAsync` requires the
> **Claude brain**, and there is no API key on this machine — so `look` never fires on the
> `home` profile regardless of hardware.~~
>
> **Struck 09/01/2026. Half of that was wrong and the wrong half did the damage.** The
> camera *has* since been opened — a 1280×960 frame at brightness 0.57, off a handset. And
> there was never a missing key: `data\apikey.dat` decrypts under this account, and the
> `cloud` profile sets `Brain: claude`. **`look` never fires on `home` because `home` is a
> local brain**, which is the first clause of that sentence and the whole of the answer. The
> second clause was a guess that read like a finding, and it stood for four versions. See
> v0.24.1 in [[Changelog]].

## Stage 10 — The interface she is operated through

Every stage until now has improved *her*. This one improves the surface a person touches,
and it is deliberately last: the control surface can only be designed once it is known
what she finally does. Music brings a now-playing state, Stage 8 brings a second renderer,
Stage 9 brings a camera permission and a presence indicator. Designing the chrome before
those exist means designing it twice.

**What is actually wrong today.** The face page grew a control at a time, which is the
honest way to build but shows:

- **The console row has no hierarchy.** Talk, the text field, Hush, Log, Settings and
  Health are six flat siblings, four of them identical `ghost` buttons. Nothing tells the
  eye that the microphone is the primary action and the other five are not.
- **The meta strip does two jobs.** Voice / Ears / Brain / Profile is a status readout —
  and the API key input sits in the same strip. A settings control living in the status bar.
- **Three drawers, three hand-written headers.** Transcript, Settings and Diagnostics each
  re-implement the same title bar, close button and empty state. One component written
  three times, so a change to drawer behaviour is three edits and a chance to miss one.
- **`face.css` has no tokens.** 231 lines of hand-tuned values, with the day-cycle palette
  living in the shader and the UI palette living in the stylesheet, unaware of each other.

**The brief, which is not "make it prettier".** She is meant to be always-on, on a desk or
a wall, *read from across the room*. What she is wearing now is laptop chrome: 13px labels,
a status strip that has to be leaned into, and a hierarchy tuned for a mouse. The primary
input is a voice; the mouse is the fallback. The interface should say so — state legible at
distance, controls that get out of the way while she is talking, and the whole thing
readable at a glance rather than studied.

**The work:**

- **Tokens first.** Pull color, type scale, spacing and radii out of `face.css` into custom
  properties, and let the room's time-of-day palette drive the UI's — the chrome should sit
  in the same light she does, not float above it in a fixed grey.
- **A real component set**: drawer, field row, status pill, icon button, primary action.
  Written once, used everywhere. This is the thing that makes Stage 8's second renderer
  cheap, because an Unreal face will need the same controls built to the same language.
- **Re-lay the console** around one primary action, with the secondary controls demoted and
  the status readout separated from the settings that change it.
- **States, not just screens.** Idle, listening, thinking, speaking and *failed* each need a
  designed appearance. The failure state is the one that always gets skipped, and it is the
  one an end user actually meets.

**Fix the loop before doing the work.** Her UI lives inside WebView2, so iterating on it
currently costs a rebuild, a relaunch and a screenshot driven through UI Automation —
minutes per change, which is enough friction to make anyone stop refining early. The
components get built and reviewed as a standalone page first, then ported. That is also
the shape [Claude Design](https://claude.ai/design) wants: a design-system project, synced
with `/design-sync`, needing a one-time `/design-login` from an interactive session. Worth
it if the design language is meant to outlive this app — which the Stage 8 argument above
says it is. Not worth it for a single console.

**The architecture rule still holds.** The chrome is part of the renderer. Anything this
stage wants from the host — a now-playing string, a permission state — goes over the
protocol as an additive, versioned message, never as a special case wired around it.

**Exit test:** stand at the far side of the room and know, without leaning in, what she is
doing and what she last said. Then add a new setting and find it takes one line, because
the field row already exists.

### Where this actually got to, 08/30/2026 — read this first on the new machine

The mock was approved and **all six decisions were built and are in the tree**. The work
was interrupted partway through the follow-up items, so this is the honest state.

**Landed and verified in the browser:**

- **Tokens.** Colour, type scale, spacing and radii are `:root` custom properties in
  `face.css`. Every value in that file now comes from one place.
- **One drawer, four tabs** — Transcript, Settings, Health, Dev — replacing three
  hand-written drawers with their own headers and close buttons.
- **The API key moved into Settings.** A missing key lights an amber pill in the status
  strip that opens Settings with the field focused.
- **Hush is transient**, inside the text field, shown only while she is thinking or
  speaking. Esc still works and also closes the drawer.
- **The chrome sits in her light.** The day-cycle keyframes gained `chrome`,
  `chromeAlpha` and `chromeInk`; the environment hands them to the page as
  `--room-tint`, `--room-ink` and `--room-line`. Header, console and caption move
  through the day with the wall.
- **The caption reaches distance size** — up to ~34px against the old 25px ceiling.
- **Status pills carry health dots** (green / amber / grey) and the strip holds no
  controls at all any more.
- **"Listening Post" is now "In residence"** — true today, and still true when she runs
  the house, which "House systems" would not be until Home Assistant lands.
- **The bust has a mouth.** It never really did: the aperture closed to 1.6% of its
  height and sat 0.076 *inside* the head, so she only had a mouth while speaking. It is
  now two deep lip forms whose centres sit inside the skull with only their front caps
  emerging — they taper at the corners exactly as the head curves away — plus a
  recessed dark aperture. Her **eyes were broken the same way**: the iris reached
  z ≈ 0.839 against a face surface at 0.871, so it was buried. Both are fixed.
- **A VRM no longer takes the full room key.** These models are authored for roughly
  unit lighting and this room runs to 2.2 at midday, which clipped her face to a white
  oval. `setLightScale` on the avatar interface scales her material response instead of
  dimming the room, so the bust and the wall are untouched and she still darkens at
  night. Rim lighting is off and the MToon outline is widened.

**Caught on the way in:** the palette callback fires *during* `createEnvironment`, before
`avatar` is initialised — a temporal-dead-zone error that stopped the whole scene
building. The `ready { faceBuilt: false }` signal from Stage 3 is what surfaced it.

### Still to do

1. **The VRM's textures never load.** This is the real reason her eyes, brows and mouth
   are hard to see, and it is *not* the lighting — the lighting fix above was worth
   making but is not the cause. The face materials report `hasMap: false` and the
   console logs `GLTFMToonMaterialParamsAssignHelper: Failed to load texture` eight
   times on load. Strong suspicion: the sample model carries **KTX2 / Basis-compressed
   textures** and no `KTX2Loader` is registered on the `GLTFLoader` in `vrm-avatar.js`.
   Register one and confirm before tuning anything else about her face.

   **Update 08/30/2026 — the model is gone, and the theory is now untestable.** The
   `.vrm` did not survive the move: `%APPDATA%\Octavia` was recreated from scratch on the
   VM at 18:32 that day, and there is no `.vrm` anywhere on either machine. **No note in
   this repo or the vault ever recorded which model it was**, so it cannot be re-fetched.
   Worth knowing: the v0.6.1 screenshot shows her *already* untextured, so this was never
   a regression — she has not once rendered with textures.

   Half the theory is confirmed: `vrm-avatar.js` builds a bare `new GLTFLoader()` and
   registers only `VRMLoaderPlugin`, so **`setKTX2Loader` is never called** and the
   vendored `GLTFLoader`'s `ktx2Loader` stays null. A KTX2 model *would* fail exactly as
   observed. Whether the lost model actually used KTX2 can no longer be established.

   Two replacement models are now in `%APPDATA%\Octavia\avatars`, and **neither uses
   KTX2** — both carry plain PNG, so they should render as-is and will not reproduce the
   fault:
   - `AvatarSample_A.vrm` — the VRoid sample, **VRM 0.x** (`extensionsUsed:
     ["KHR_materials_unlit","VRM"]`), free for any use without credit.
   - `VRM1_Constraint_Twist_Sample.vrm` — pixiv's own three-vrm sample, **VRM 1.0** with
     MToon and 19 PNG textures; the closer match to the expression vocabulary we target.

   So the next step is no longer "register a KTX2Loader" but **"load one of these and see
   whether she is textured at all"**. If she is, the fault was the old model and the
   loader gap is a latent bug worth closing anyway. If she still is not, the cause is in
   our own loader setup and KTX2 was a red herring.

   **RESOLVED 08/31/2026 — it was the CSP, and KTX2 was a red herring.** `img-src` did not
   list `blob:`, and neither did `connect-src`. glTF carries its textures inside the
   binary; three.js decodes them to a `Blob` and loads them from a `blob:` URL, so *every*
   texture in *every* model was blocked regardless of format. That is why she had never
   once rendered with a face. Adding `blob:` to both lists fixed it: 20 of 28 materials
   textured, zero errors, verified in the browser and in WebView2. The missing
   `setKTX2Loader` is still a genuine latent gap for a model that needs it — but it was
   not this, and neither was the lighting.
2. ~~**Local-first profiles.**~~ **Done 08/31/2026, v0.19.3** — and it was not cosmetic.
   The live `config.json` on this machine said `Profile: "live"` → `Brain: "claude"` with
   **no key stored**, so every turn would have been refused with "No API key yet". She only
   worked because the shortcut passes `--profile dev`; started any other way she was mute.
   That is also where the API-key nag came from.

   `home` (local, `large-v3-turbo`), `cloud` (Claude) and `dev` (local, `small.en`) are the
   shipped profiles, `Profile` defaults to `home`, and **base `Brain` defaults to `local`** —
   which is the part that matters, because an unnamed or misspelled profile falls back to the
   base and a keyless Claude is the worst thing to fall back to. `live` still resolves, so
   older config files are untouched. An undefined profile is now a **warning** naming the
   ones that exist; it used to be an info line, which is how this went unnoticed.

   Verified by launching her with no `--profile` at all: `profile 'home' (config file):
   brain=local` and `brain: qwen2.5:7b-cpu (local)`.
3. **Docs and vault for the Stage 10 work.** *Mostly done 08/31/2026, v0.19.3.*
   `README.md` now describes the drawer and its tabs, the typing button, the status-readout
   setting, and the local-first default — its opening line and the "two brains" section both
   claimed Claude was her mind, which stopped being true this release. `PROTOCOL.md` turned
   out to already document `camera`, in both the `hello` table and its own `look`/`sight`
   section, so that item was already closed. The vault gained *Chrome over the room* in
   [[The Room]] at 0.19.1.

   **Screenshots: closed differently than written, 08/31/2026.** Shots for v0.10.0 itself
   cannot be taken — that console no longer exists. What was built instead is the thing that
   stops the gap reopening: `tools\shoot.ps1` captures her real window at the 1100x780 every
   existing shot uses, named from the version in the csproj so it cannot be misfiled;
   `tools\poke.ps1` clicks and scrolls in window coordinates so a set can be *retaken* after
   the chrome moves; and `check-vault.ps1` reports when the current version has no shot.
   Six were taken for v0.19.3, and the two orphaned files already in the vault (v0.12.0,
   v0.15.0) were finally written up. The standing rule is now **a look after every version
   bump**, in the same change set as the code — see [[Screenshots]].
4. **Re-take the Stage 10 exit test** on the new machine, from an actual sofa.

---

## Stage 11a — She should hear the room, not just the machine *(found 08/31/2026)*

**Her music sense taps WASAPI loopback, which is what *this computer* is playing.** Music
from a speaker in the same room — another PC, a phone, a hi-fi — never reaches it. It
reaches the microphone, and the microphone is not wired to the analyser at all.

Found the obvious way: she would not dance, and the loopback was open and reporting zero
while music was plainly audible in the room. The mic read 0.013 against a 0.004 noise
floor, so it *could* hear it.

For a thing that lives in a room rather than on a desktop, this is the wrong way round.
The design is sound for "react to what I am playing"; it is silent for "there is music on".

**The shape of the fix.** `MusicAnalyzer` already consumes arbitrary float PCM, and
`WhisperRecognizer` already has mic frames at 16 kHz mono. Tee them into a second analyser
instance and she has both sources. Two things need care:

- **The gate.** Music in the room already risks false wakes; lyrics are speech. Anything
  here must not make that worse, and might be able to make it better — knowing music is
  playing is exactly the context a gate could use to be *more* sceptical.
- **Crest, again.** Room audio through a boom mic is reverberant and gain-controlled, so
  the beat will be far less clean than loopback. `EarsTest music` should be able to
  measure the mic source too before anyone trusts it.

Cheap, high value, and it makes her feel like she is in the room rather than in the PC.

**Built in v0.16.0** as `MusicFromRoom`, off by default. A second `MusicAnalyzer` fed from
the frames the voice detector already reads. **Not yet verified against real room audio** —
**Verified 08/31/2026**: music on another computer, loopback reading energy 0.00, and she
reported 141 bpm at confidence 0.49 through the microphone. The lower confidence against
loopback's 1.00 is the room-and-boom-mic penalty, showing up exactly as predicted.

## ~~Stage 2a — The streaming loop blocks a thread on every token~~ *(done 08/31/2026)*

**Fixed in v0.19.3.** `while (await reader.ReadLineAsync(cancel) is { } line)` — the peek is
gone rather than worked around, and the build is at zero warnings. The audit the entry asked
for came back clean: `EndOfStream` appeared exactly once in the whole project, and the other
synchronous waits (`App.xaml.cs`, `SystemReport`, the dispose path) are all on threads that
have nowhere else to be.

**`LocalBrain.cs:96` is `while (!reader.EndOfStream)`, and `EndOfStream` is a synchronous
read.** The compiler says so — CA2024, the one warning in an otherwise clean build — and it
is right in a way that matters more here than it usually would.

`EndOfStream` has to peek at the underlying stream to answer, and on a server-sent-event
response from Ollama there is nothing to peek at until the model produces the next token.
So the loop blocks a thread pool thread waiting for it, then hands the line it was already
waiting for to `await reader.ReadLineAsync` on the very next statement. The `await` is
decorative: the blocking already happened.

This costs more on this machine than it would have on the VM. The brain is pinned to the
CPU deliberately (see the Modelfile in [[Changelog]] 0.14.x), tokens arrive slowly, and a
thread is parked for the entire length of every reply rather than for a moment.

The fix is one line and removes the peek rather than working around it:

```csharp
string? line;
while ((line = await reader.ReadLineAsync(cancel)) is not null)
{
    if (cancel.IsCancellationRequested) break;
    ...
}
```

Worth doing at the same time: **check whether anything else in the project reads a stream
this way.** CA2024 fires per call site, and one warning in the build output is easy to stop
seeing — which is how this one survived several releases.

## ~~Stage 10a — Nothing checks the face's own syntax~~ *(done 08/31/2026)*

**Both halves landed in v0.19.3.**

`SyntaxChecks` takes the second option this entry proposed — it loads the real page in the
WebView2 the app already depends on and lets Chromium do the parsing. No hand-rolled parser,
no Node, and the same engine that will run it for real. It checks the virtual `https://`
origin too, so the CSP behaves exactly as it does in the app rather than as a `file://` load
would. It asserts three things: nothing raised a `SyntaxError`, `window.Face` was published,
and the bridge would have sent `ready`.

**Proved by breaking it.** Appending an orphan `});` to `bridge.js` — the v0.18.0 fault
exactly — turns it red and names the file and line: `SyntaxError: Unexpected token '}'
(https://octavia.face/bridge.js:848)`, with `window.Face` still ok and `ready` missing, which
is the precise signature of that bug. A check nobody has watched fail is not yet a check.

The host half is `MainWindow.WatchForFace`: 30 seconds after navigation, if the session has
never seen a `ready`, it logs an error and shows the existing fallback panel — a surface that
does not depend on the renderer working, which is the whole point. The grace is deliberately
many times longer than needed, because `ready` is sent when the socket opens rather than when
the scene finishes, and the cost of being wrong is hiding a working face.

*Run it alone with `dotnet run --project tools/EarsTest -- syntax`.*

**A JavaScript syntax error in `wwwroot` is invisible to everything this project has.**
`dotnet build` does not read those files and `EarsTest` does not parse them, so the build
goes green, every check passes, and the face is simply dead.

Found the hard way in v0.18.0: a `sed` deleting four lines by number took the wrong four,
left an orphan `});` in `bridge.js`, and the whole bridge failed to parse. The symptom was
that the drawer button stopped opening the drawer — nothing else, no error anywhere a
build would show it. It was caught by opening the browser console, which is exactly the
tool that does not exist on someone else's machine.

Every other silent failure in this project has been given a voice; this one has not. The
fix is small:

- **Parse every `wwwroot/*.js` in `EarsTest`.** No Node on this machine, so either shell
  out to whatever is present, or — better, and dependency-free — load each file in the
  WebView2 that is already a dependency and let it report the parse error.
- **Report `faceError` louder.** The face already sends unhandled errors to the host, but
  a file that never parsed never runs the code that would send them. The host could notice
  that `ready` never arrived and say so.

The second is the more valuable half: it turns "she looks broken" into a line in the log
for *any* fatal renderer failure, not just a syntax error.

## Stage 11 — The room, the props and the chrome *(agreed 08/31/2026)*

Four pieces of polish that the textured face made visible, because until v0.12.0 nobody
could see her well enough to notice.

- **A loading splash.** She currently shows a finished-looking console the instant the
  window opens, while the WebView2 environment, the face socket, the scene, the voice and
  possibly a model download are all still coming up. The gap between "looks ready" and
  "is ready" is where every "she ignored me" report starts. Hold a splash until the
  renderer says `ready`, and let it name what it is waiting for rather than spinning
  anonymously — the same argument as the diagnostics stage, one screen earlier.

> **Still not right after four attempts — revisit properly. (08/31/2026)**
>
> Height is solved: the cups sit on the eye line, taken from the `leftEye`/`rightEye`
> bones, and that part is correct. **Width and depth are not**, and adjusting them by eye
> did not converge — each round fixed the last complaint and introduced another.
>
> Two things learned that the next attempt should start from rather than rediscover:
>
> - **`Box3.setFromObject` on a skinned mesh returns its *bind pose*.** The body is one
>   mesh whose bind pose is a T-pose, so it measures ±0.69 wide and reaches above the jaw.
>   Any "sample the vertices near the head" approach collects outstretched arms unless it
>   filters by material first.
> - **The face mesh (`Face_00_SKIN`) is half-width 0.109 on the sample model**, against the
>   0.164 currently used — roughly 50% too wide. But simply using the measured skull put
>   the cups *inside* her hair, because on a long-haired character they belong outside it.
>   The missing quantity is the hair silhouette at ear height, not the skull.
>
> So the real fix is probably to measure the **hair** meshes at ear height and clamp just
> outside them, with the skull as a floor. Worth doing with the model in front of you and
> a way to nudge the numbers live, rather than a rebuild per guess — a small dev-panel
> slider for cup width and depth would pay for itself in one sitting.

- **The headphones do not sit right.** They are attached to the head bone and sized from
  `headPoint.y * 0.115`, which is a guess at head *width* derived from character *height*.
  A VRM does not standardise head size, so the guess is wrong per model. Take the head
  bone's actual bounding box instead, and offset along its local axes rather than the
  world's. See [[The Avatar Interface]].
- **The background is static.** It is a flat gallery wall with a day-cycle tint. Give it
  depth: a parallax layer or two behind her, subtle drift, and an audio-reactive mode fed
  by the `music` message that already exists. Cheap on GPU, and this machine's GPU is the
  weak half — so measure, and keep it switchable.
- **The console is sized for typing, and she is for talking.** The text field dominates
  the bottom of a full-screen window when most turns will never use it. Collapse it behind
  a button, and stack the status line bottom-left rather than spreading it across the
  full width. This is a 10-foot interface first and a form second.

## Stage 12 — Hands: Home Assistant, UniFi, and an integration seam

The point of the whole project. She should run the house, and that means talking to
things that are not her.

**Design it as one seam, not N integrations.** The argument for MCP here is strong: it is
a published protocol with a tool-definition shape both Claude and local models can be
handed, it keeps each integration out-of-process behind a boundary, and it means a new
capability is a new server rather than a new branch inside `OctaviaSession`. That matches
the constraint the project already holds — the host owns capability, the face stays dumb.

Sketch:

- `IToolProvider` beside `IBrain`, so tools are discovered rather than compiled in.
- An **MCP client** in the host, speaking stdio to local servers and HTTP to LAN ones.
- Servers, each independently useful and independently broken-able: **Home Assistant**
  (its REST and WebSocket APIs are well documented and token-authenticated), **UniFi**
  (network health, who is home by device presence), and later whatever else.
- **The gate becomes load-bearing.** A model that can turn the lights off must be much
  more certain it was addressed than one that can only talk. Expect the confirmation
  rules to need a second look before anything can act.
- **Nothing destructive without confirmation**, and a written allow-list of what she may
  do unattended. This is the stage where a mistake stops being a wrong answer and starts
  being a dark house.

### Where this got to, 08/31/2026

**The seam is built and tested. She cannot call a tool yet.** That split is deliberate and
worth stating plainly rather than leaving someone to discover it.

Done, and proven against a real child process speaking real JSON-RPC:

- `ITool` / `IToolProvider` / `ToolRegistry` — the seam. A brain is handed a registry and
  never learns how many servers there are or which one answered.
- `McpClient` — MCP over stdio: `initialize`, the `notifications/initialized` the spec
  requires, `tools/list`, `tools/call`, newline framing, a 30-second per-request timeout,
  and a read loop that fails every pending call when a server dies rather than hanging.
- **`ToolRisk`, and the rule that dangerous tools do not run unasked.** MCP carries no
  risk annotation, so it is inferred from name and description and deliberately biased
  towards asking: a read misjudged as dangerous costs one question, an unlock misjudged
  as safe costs rather more.
- `McpServers` in config — command, args, env, enabled. **Tokens go in `Env`**, because an
  argument is visible in the process list to every account on the machine.
- `tools\mock-mcp.ps1` — a three-tool server (a read, an act, an unlock) so the seam is
  testable on a machine with no house attached. `EarsTest` drives the real client against
  it: 11 checks, including that the unlock is refused unconfirmed and runs when confirmed.
- The session starts servers in the background, logs what they offer, and reports them in
  `hello`, so "is the integration actually connected" does not need a log to answer.

**Not done: the brain-side tool loop.** Collecting `tool_use` blocks out of a streaming
response, running them, appending `tool_result` and re-requesting. It was left rather than
written blind for two reasons — it changes the main conversation path that currently works,
and ~~there is no API key on this machine to verify it against~~. Writing it untested and
calling it done would repeat exactly the mistake this version spent its time undoing.

> **One of those two reasons was never true** *(corrected 09/01/2026)*. There is a key, and
> `--profile cloud` reaches Claude; the belief that there was not came from a stale note
> after the machine move and propagated into three roadmap entries. **The first reason still
> stands entirely** — this changes the path every turn takes — but the second no longer
> excuses anything, and "we cannot verify it" was doing most of the work in that sentence.
> Whoever picks this up can now write it *and watch it run*.

**When it is written**, the safe shape is: build the request identically when the registry
is empty, so a machine with no servers configured is byte-for-byte unaffected.

### It is written — **the loop is built, 09/02/2026, v0.29.0**

She was asked *"what hardware is on my network right now?"* and answered from the gateway:
*"There's a Dream Machine Pro SE acting as your gateway, and a UniFi AC Pro access point."*
Nobody named a tool. **That is Stage 12's whole point reached** — the seam, a real server
and the last hop, all connected.

**The safe shape above was followed literally.** `Ask()` builds the request with two separate
object initializers rather than one and a conditional assignment, so *"identical when there
is nothing to offer"* is something a reader can check instead of something they have to
trust a serialiser about. No `Tools`, no reordering, no extra block — the system-prompt cache
breakpoint is untouched for everybody with no servers configured.

How it works, and what each decision cost:

- **Tool calls are assembled from the stream**, not from a second non-streaming request.
  `content_block_start` opens a call, `input_json_delta` fragments accumulate into its
  arguments, and nothing is parsed until the block ends. She keeps speaking while it happens
  — *"Let me check that for you"* was spoken before the call went out, which is the whole
  reason for streaming and would have been lost by the simpler design.
- **Four rounds, then she stops and says so in the log.** The failure this guards is a model
  answering its own tool result with another call forever: it costs money every lap and,
  unlike a slow answer, never ends on its own.
- **The tool exchange is not written to history.** `Conversation` holds strings, and widening
  it to structured blocks would change every brain and the diagnostics bundle. Whatever the
  tools said is inside the sentence she just spoke, so the next turn keeps the substance and
  loses only the ability to quote the raw result. When that matters it is a real change and
  belongs on its own.
- ~~**`confirmed` is always false**, and that is the honest state of it rather than an
  oversight. Carrying a spoken *yes* from one turn to the next is its own piece of work.~~
  **Built, v0.32.0.** *"Unlock the front door"* → *"Do you want to unlock the front door?"* →
  *"yes"* → unlocked, with the log showing `needs confirmation; not run`, then `confirmed by
  the last thing said`, then the call. *"No"* produces the refusal and nothing else.

  **Consent lasts exactly one turn and is to a *call*, not a tool.** Read and cleared at the
  top of every turn, so a question is answerable by the very next thing said and nothing
  after it. The arguments are part of it: *"yes"* about the back door is not consent to
  unlock the front one, however the model phrases its second attempt.

  **The rule refuses when unsure**, because the costs are not symmetric — a yes misread as a
  no costs one repeated question; a no misread as a yes costs whatever the tool does.
  Agreement must be present *and* disagreement absent, so *"yes, but not yet"* is not
  consent. A consent that survived several turns would let a yes about something else open a
  door, and *"say yes to the next thing she asks"* is a sentence a television can produce.

  The refusal text changed with it, and earns its place: told only that confirmation is
  needed, a model will often ask and answer itself in one breath — *"shall I unlock it? Yes,
  unlocking now"* — which is a confirmation nobody gave. It now says to ask plainly, in one
  short question, and say nothing else.

**Proven by a probe, never by the suite.** `EarsTest -- toolloop` asks two real questions
against the real API and the real gateway. It is deliberately *not* in the default run:
*"a self-test that spends money is a bad self-test"* is already written down here, and this
one spends money every time. Everything up to the last hop stays covered for nothing by
`ToolChecks` and `UnifiChecks`.

~~**Claude only, so far.**~~ **Both brains, v0.29.1** — written the same day, because the gap
it left was the one that mattered. The `home` profile is a local brain, so *"she can use
tools"* and *"she can use tools on the profile she is actually run under"* were different
claims, and only the second one is worth anything day to day.

`LocalBrain` now sends an OpenAI-compatible `tools` array and assembles `tool_calls` out of
its own stream. **qwen2.5:7b answers correctly**: *"Your network has a UniFi Dream Machine
PRO SE and an AC Pro access point."*

Two things the shape forced, neither cosmetic:

- **The index identifies a call across chunks, not the id.** Ollama sends a call whole in a
  single chunk; the OpenAI streaming shape lets arguments arrive as fragments, and the id
  may appear only in the first. So every field accumulates — free when there is one
  fragment, correct when there are twenty. Written against the format rather than against
  the one server that was to hand.
- **A machine with no tool servers must not start sending a `tools` key.** Some servers
  refuse it outright when the model has no tool template, so *identical when there is
  nothing to offer* is a harder requirement here than for the hosted brain — and it is met
  the same way, with two request shapes rather than one and a conditional key.

**A 7B model embellishes.** Asked about hardware it added *"with their firmware versions up
to date"*, which no tool said. This is the small-model problem in a new place: the tool
result is exact, and what a small model does with it is not. Nothing is wrong with the loop
— it is an argument for keeping dangerous tools behind confirmation no matter which brain is
driving.

**Then Home Assistant and UniFi are configuration, not code.** HA ships an MCP server of
its own; UniFi has community ones, and failing that a small server is a day's work against
its API. That is the whole return on doing the seam first.

### The house, as it actually is *(answered 08/31/2026)*

**No Home Assistant. Smart devices are on Google Home. UniFi is a UDM SE at `10.1.1.1`.**

That changes the recommendation, because **Google Home has no usable API for a Windows
desktop service.** The Home APIs Google publishes are mobile SDKs for Android and iOS
apps; the Smart Home API is for device *manufacturers*. There is no supported way for a
program on this PC to say "turn the kitchen light off" to Google Home.

So the recommendation is **install Home Assistant, and let it be the only thing she talks
to.** Not as a preference — as the thing that makes the stage possible at all.

- **Most Google Home devices are not really Google's.** They are Matter, Thread, Zigbee or
  plain WiFi devices that happened to be enrolled through Google. Matter is explicitly
  multi-admin: HA can control them *alongside* Google Home, locally, with Google still
  working exactly as it does now. Nothing has to be torn out to try this.
- **HA ships an MCP server** (2025.x onwards), so it plugs into the seam already built
  with no integration code at all.
- **HA has a UniFi integration**, which folds the UDM SE into the same surface — one
  server to configure rather than two, and network presence becomes just another sensor.
- For anything genuinely Google-only, HA's Google Assistant SDK integration can relay a
  spoken command. Clunky, and a fallback rather than the plan.

**Where to run it:** this machine has the cores and the memory, so HA in Docker or a VM
here is the cheapest start. A Pi or a spare box is tidier long-term — HA does not enjoy
sharing a machine that reboots for games.

**The order that follows:** install HA → adopt whatever it can see → enable its MCP server
→ add one `McpServers` entry → *then* write the brain-side tool loop, with something real
to call.

### UniFi came first, and needed no Home Assistant at all *(09/02/2026, v0.28.3)*

The recommendation above is still right about the *house*. It was wrong about the network:
**the UDM SE answers an official local API of its own**, so `tools\unifi-mcp.ps1` plugs into
the seam with no HA anywhere in the picture.

- **UniFi Network Integration API v1**, application `10.6.101`, at
  `https://10.1.1.1/proxy/network/integration/v1`, authenticated with an `X-API-KEY` made in
  the UniFi UI. Sites, devices, clients, per-device statistics.
- **UniFi Protect answers the same key**, application `7.2.105`, at
  `/proxy/protect/integration/v1` — cameras, and a JPEG snapshot per camera.

Five tools, every one a read: `list_devices`, `list_clients`, `get_status`, `find_client`,
`list_cameras`. **`list_clients` is presence** — eighteen named clients, `Kitchen - Plug -
Microwave` and the like, which answers "is anyone home" without a single smart-home
integration existing yet.

> **A 401 is not proof that an endpoint exists.** The first probe read `401` on the
> integration path and took it as confirmation the API was enabled; a nonsense path under
> `/proxy/network/` returns `401` too, because the proxy answers before it routes. Only a
> real key settled it. *An error that is produced before the question is understood tells you
> nothing about the question.*

**Both Protect cameras are offline** — `Front Door` and `Back Garden`, both `DISCONNECTED`,
and a snapshot returns `503` rather than an image, so the state is real rather than a stale
flag. `list_cameras` therefore reports what is reachable rather than what exists, because
"she has cameras" and "she can see anything" are separate claims.

~~**Not built: the snapshot tool.**~~ **Built 09/02/2026, v0.31.0**, once both blockers went:
the loop landed in v0.29.0, and the owner switched a camera on. `look_at_camera` returns an
MCP image block, and she describes what is in it.

**The seam was text-only and is not any more.** `IToolProvider.CallAsync` returned a string,
and `McpClient` carried a comment saying an image block *"would need the vision path and is
left for whenever that matters"*. A camera is when it matters, because the useful answer to
*what is at the gate* is not a sentence about a JPEG. `ToolAnswer` carries text and an
optional image; every text-only path reads exactly as it did, through an implicit conversion.

**An image-bearing answer must still say in words what it captured**, which is a rule rather
than a nicety: `LocalBrain` has no eyes, so it takes the text and logs that it dropped a
picture — a local turn stays usable instead of blank. Only the first image is kept; a tool
returning twelve frames would otherwise put twelve into one turn, which is a bill rather than
an answer.

> **What she said on the first real look.** Asked to *"have a look outside and tell me what
> you can see"*, she reported that the Back Garden camera *"seems to have been moved or
> knocked, it's actually pointing at the inside of a room right now, showing a ceiling fan
> and a wardrobe"* — which the frame confirms exactly.
>
> She described **what was there rather than what the camera was called**, and flagged the
> mismatch herself. That is the entire argument for handing a model the picture instead of a
> description of the picture, and it arrived unprompted on the first attempt.

The local brain declined honestly rather than bluffing — *"I can't look outside yet, but I can
check the cameras. Which camera would you like me to check?"* Not ideal, and not a lie either.

**Network and Protect are one server, not two**, which departs from "each integration
independently broken-able" above. They are two applications on one appliance behind one
address and one key: when the UDM is unreachable both are, so there is no independence to
preserve. If Protect ever moves to its own NVR, splitting it is a copy and a config entry.

`EarsTest -- unifi` drives the real gateway and **skips** when no key is configured, so it
stays green on a machine with no house. Eight assertions, and the one worth having is
*"every tool is judged a read"*: `RiskOf` looks for its dangerous words first, so a
description that gained a `restart`, a `reset` or an `order` would quietly turn a status
query into something she stops to ask permission for, and nothing else would notice. Broken
on purpose to watch it go red.

~~**Still not built: the brain-side tool loop.** She lists five tools and can call none of
them.~~ **Built the same day, v0.29.0** — and the argument for doing UniFi first held: the
loop was written and watched running against read-only tools with nothing in the house to
break. See Stage 12 above.

## Stage 13 — Away: a phone that asks the house how it is

"How is everything at home?" from somewhere else. This is a **second face over the Stage 3
protocol**, which is exactly what that protocol was built for — an Android client is a
renderer plus a microphone, not a second Octavia.

The honest problem is not the app, it is exposure: the face socket is loopback-only with a
per-run token, deliberately. Reaching it from a phone means Tailscale or Wireguard back to
the house — not a port forward. Decide that before writing any Kotlin.

### What this stage actually needs, in order

**None of it is the app.** An Android client is a WebSocket, a microphone and a renderer;
the work that makes it possible lives in this repo and in the network, and doing it in the
wrong order produces a phone app that can only be used on the sofa it was built on.

1. **A transport that can leave the machine.** `WebSocketFaceServer` binds `127.0.0.1`
   deliberately (see Stage 3 — it avoided needing a urlacl and elevation). A second
   binding on the LAN address is small, but it turns a private socket into a listening
   service and everything below follows from that.
2. **Authentication that survives a restart.** The per-run token is regenerated every
   start, which is correct for a page the host itself loads and useless for a phone in a
   pocket. A paired-device secret, stored per device and revocable, is the minimum.
3. **The network decision, made once and written down.** Tailscale or Wireguard, so the
   socket is never reachable from the internet at all. **Not a port forward** — a
   microphone and a house controller behind a forwarded port is the worst version of this
   project.

   **Decided 08/31/2026: the UDM SE's own WireGuard server. Tailscale is not needed.**

   The UDM SE runs a WireGuard VPN server natively. A phone connects to it, receives an
   address on the LAN, and can then reach `10.1.1.x` — including this PC — directly. No
   software on the PC, no third-party coordination service, and Octavia's socket is never
   exposed beyond the LAN.

   The one forwarded port this needs is **WireGuard's own UDP port on the router**, which
   is a completely different proposition from forwarding hers: a WireGuard endpoint does
   not answer unauthenticated packets at all, so it is silent to a scanner. That is the
   distinction the warning in `WebSocketFaceServer` is about — not "never forward
   anything", but "never forward *her*".

   Tailscale remains the fallback for one specific case: **CGNAT**, where the ISP gives no
   reachable public address and no port can be forwarded at all. Worth checking before
   setting WireGuard up.

   **Do not forget the Windows firewall.** Binding every interface is necessary and not
   sufficient — inbound TCP on the face port from anything but loopback is blocked by
   default, so `RemoteAccess` alone will look broken. One inbound rule, scoped to the LAN
   subnet rather than Any.

   > **As it actually stands, 09/01/2026.** `RemoteAccess` is **on**, she answers on
   > `10.1.1.21:8848`, and the Windows firewall on this machine is **off entirely** — so
   > the remote key is the only thing between the LAN and her microphone. Until v0.23.1
   > that key could never match, which meant the socket was open and nothing could get in:
   > accidentally fail-closed. It is a real lock now, which makes the firewall being off a
   > choice worth revisiting rather than a harmless one. A scoped inbound rule is still the
   > right shape; turning the whole firewall off to get one port is not.
4. **A protocol subset for a phone.** A phone does not want visemes at 60 Hz. `hello`
   already carries capabilities; the client should be able to say what it wants and be
   sent only that. This is a protocol change and belongs in PROTOCOL.md before any client
   depends on it.
5. **Then the app**, which at that point is genuinely small: connect, hold a mic button,
   stream the reply, and show the house's state from Stage 12's tools.

Worth saying: **step 5 is a separate project and a separate repository.** It needs the
Android SDK and a device to test on, neither of which is what this repo is. The value of
listing 1–4 is that they are all doable here, and they are what makes the app a week
rather than a month.

That repository now exists: `C:\Projects\Octavia-Android`, private on GitHub, `minSdk 28`.

### Step 1 was half done, and the missing half was the page *(closed in 0.20.0)*

Binding the socket to the LAN was recorded here as "a transport that can leave the
machine". It was the transport and not the face. **`wwwroot` was never served by anything**
— it reaches the built-in page through a WebView2 virtual host mapping, which is a feature
of that control rather than a server, and nothing in this process had ever answered a GET.
A phone could therefore open the socket and still have no page to run inside it.

Vendoring the page into the client was the obvious alternative and it lost on a fact: there
are **two** virtual host mappings, and the second serves her avatars folder. A VRM is user
data, in a git-ignored folder, chosen at runtime, in no repository — it can never be baked
into a client, so the host had to serve files either way. The real choice was one mechanism
or two of them drifting.

So the socket now answers plain GETs from `wwwroot` and `/avatars/`, sharing one port and
one origin with the WebSocket, with the credential carried in a cookie because a page's
sub-resources cannot carry a query string. See `PROTOCOL.md` → *Serving the face*.

**Two things this uncovered that Stage 14 has to carry:**

1. `getUserMedia` does not run on a plain `http://<lan-ip>` origin — it is not a secure
   context. So a tablet's camera and microphone cannot live in the WebView. The Android
   client owns them **natively** and answers `sight` itself; the WebView stays a renderer,
   which is what the protocol always said a face was.
2. The avatar URL had to be rewritten in the *renderer*, because one `hello` is serialised
   once and broadcast to every face — the host cannot say something different to each. That
   is item 1 of Stage 14 (faces have no identity) drawing blood for the first time.

## Stage 14 — More than one body *(agreed 08/31/2026)*

Stage 13 imagined a phone that *asks how the house is* — a thin, away-from-home client that
mostly reads. **This is a bigger ask and it deserves its own stage:** a tablet that is a peer
to the desktop face, open at the same time as it, with her mic, her camera and her voice
working there exactly as they do here.

The difference matters because Stage 13's design survives having one weak client. This one
does not: the moment two faces can both *speak*, the host needs to know which one it is
talking to, and it currently cannot tell them apart at all.

### What already works, and it is more than expected

- **Two faces at once, rendering.** `FaceHub.Send` serialises once and fans out to the
  WebView2 page and every socket client in `_faces`. Open the tablet beside the desktop and
  both already show her, in step, today. This was the point of Stage 3 and it landed.
- **The camera is already in the right place.** `camera.js` opens the camera *in the face*,
  and says why in its own header: "it puts the camera where the *person* is — a face on a
  wall tablet has one; the machine under the desk may not." `look`/`sight` is a face→host
  round trip, so a tablet answering with its own camera needs **no protocol change**.
- **`subscribe` / `skip`** already lets a tablet decline what it cannot use.
- **`remote.key` and the LAN binding** already let it connect at all.

### ~~1. Faces have no identity — everything else depends on fixing this~~ *(done 08/31/2026, v0.21.0)*

> **Landed.** Built from [[Stage 14 - Face Identity]], a specification written on the Android
> side by the client that needed it. `FaceId`, `FaceMessage`, `Send(message, to)` with null
> still meaning everyone, `SendTo` on the socket server honouring `skip`, and `_looking`
> carrying the face it asked so an unasked `sight` is logged and dropped.
>
> **`look` no longer opens every camera.** It goes to the last face a person spoke through,
> falling back to the built-in page. That is a stopgap and is commented as one — real turn
> ownership is item 5 below and should replace it rather than absorb it.
>
> Nothing moved on the wire, so `PROTOCOL.md` is unchanged and `wwwroot` is untouched — six
> host-side files. Items 2–7 were deliberately not started; the Android side is planning
> around that split.
>
> ~~**One gap worth carrying forward:** two real cameras were never opened, because
> `MaybeLookAsync` requires the Claude brain and there is no key on this machine. The routing
> is proven at the transport level and the `sight` guard was watched firing live, but the
> camera path itself is unexercised end to end. Re-check on a machine with a key.~~
>
> **Closed 09/01/2026, from the Android side, and the stated reason was wrong.** There was no
> missing key — only a `home` profile on a local brain. `look` → `sight` walked end to end
> under item 9: `look: asking face a85b541d in room 'phone'` → `sight: 1280x960, brightness
> 0.57, spread 0.190` → `got a frame, 97 KB`, matching `CameraStill: one frame, 97 KB` in the
> handset's own log. **This sentence is why it took four versions**: it named a blocker that
> did not exist, item 9 inherited it in good faith, and nobody re-tested the claim.

`IFaceTransport` is broadcast-only in both directions: `Send(object message)` goes to
everyone, and `event Action<JsonElement>? MessageReceived` says **who sent nothing at all**.
That was a deliberate simplification — "the being stops knowing about the renderer" — and it
is exactly right for faces that only watch. It stops being right the moment a face has a
microphone.

Every item below needs a face id, so this is first and nothing else starts until it lands:

- `MessageReceived` carries the id of the face that sent it.
- `Send` gains an optional target; no target keeps today's broadcast, so no existing
  behaviour changes.
- `WebSocketFaceServer` already keys `_faces` by id, so it mostly has this. `FaceHub`,
  `WebViewFaceTransport` and `OctaviaSession` are where the work is.

Note the seam does *not* get coarser: the session learns that faces are distinguishable, not
how any of them connected.

### ~~2. Audio upstream — her ears, on the tablet~~ *(done 09/01/2026, v0.23.0)*

> **Landed**, from [[Stage 14 - A Microphone Somewhere Else]]. `IAudioSource` with a local
> and a face implementation, push-to-talk with a single floor-holder, `Flush()` on release,
> and binary frames upstream mirroring the downstream rule.
>
> **Both traps in that spec were real.** The room-music analyser was fed from
> `whisper.Audio`, so swapping the source would have moved her sense of *this* room to the
> phone with everything still appearing to work; the local microphone is now owned by the
> session, shared, and framed separately. And `WatchForSilence` would have named RDP audio
> settings at somebody holding a phone, so it is gated on `ExpectsContinuousAudio`.
>
> The "do not stop the old source" rule is a **test** rather than a comment, because it is
> exactly what a later tidy-up would undo: reintroducing `_source.Stop()` turns it red.
>
> **Needs a handset to finish:** criteria 1, 5, 6 and 9 — a held button producing a
> transcript, the desk microphone genuinely muted during a remote utterance, the room
> analyser still hearing this room, and barge-in. The seam and both traps are covered here.

<details><summary>The original plan</summary>

### 2. Audio upstream — her ears, on the tablet

`WhisperRecognizer` and `MicLevelMeter` each construct their own `WaveIn` and **are** the
capture device rather than consumers of a stream. So this is a real refactor and not a
parameter:

- Introduce an audio-source seam — `LocalMicSource` wrapping today's `WaveIn`, and a
  `FaceAudioSource` fed from a socket. Silero and Whisper stop caring where samples came from.
- A face→host `audio` message. **Binary WebSocket frames, not base64 in JSON** — base64 is a
  third more bytes and a great deal of garbage at 16 kHz.
- **16 kHz mono 16-bit PCM**, resampled on the tablet, because that is what Silero and
  Whisper want and the tablet has cycles to spare.
- Two live sources means two transcription streams. This box is a **Ryzen 7 3700X — 8 cores,
  16 threads**, not 16 cores; measure before assuming two Whisper instances are free, and
  consider one shared queue instead.

</details>

### ~~3. Audio downstream — her voice, out of the tablet~~ *(done 08/31/2026, v0.22.0)*

> **Landed**, from [[Stage 14 - Her Voice On Another Face]]. Binary frames, opt-in via
> `subscribe`'s new `want`, the format announced in `hello`, a tee at `OnAudioPlayed`, and
> per-face send queues underneath it all. `PROTOCOL.md` gained an *Audio* section.
>
> **The spec's "do this first" was wrong about why.** It claimed concurrent `SendAsync`
> throws and the catch-all silently drops a live face. Measured: 320 overlapping sends on
> one socket, nothing thrown — .NET serialises them behind a lock. A test written against
> that failure passed against the *unfixed* code. The real fault is a face that stops
> reading: un-awaited sends never complete and pile up unbounded. Same fix, honest reason.
>
> **Still open here:** acceptance criterion 2 — that the PCM plays back as intelligible
> speech at the advertised rate — has no client yet. Criteria 1, 5 and 7 are in `EarsTest`;
> 3, 4 and 6 are verified by construction and by `Hush()` raising `Finished`. The first real
> playback on the Android side is the remaining proof.
>
> Item 2 (her ears, on the tablet) is the larger half and is untouched.

**Today her voice cannot leave this PC.** `NeuralVoice` writes to a `WaveOut` and `SapiVoice`
calls `SetOutputToDefaultAudioDevice()`. A tablet in another room sees her mouth move in
silence.

There is a neat way in. `NeuralVoice.OnAudioPlayed` is *already* handed the exact PCM at the
moment it goes to the sound card — that is how the visemes stay in step with what is heard.
Teeing that same buffer to a face gets audio that is in sync with the visemes the tablet is
already receiving, for free, at the one point in the code where that is guaranteed. A
host→face `audio` message, same format, same framing.

SAPI is the harder half and can wait: getting at its PCM means `SetOutputToAudioStream`.

**Rejected: speaking with the tablet's own TTS.** It breaks the rule that a face is a
renderer, and she would have a different voice in every room.

### ~~4. Camera arbitration — `look` must name a face~~ *(landed in v0.21.0; entry was stale)*

> Struck 09/01/2026. `look` has named a face since item 1 landed — the ROADMAP simply never
> said so. Item 9 changes *how* that face is chosen, from "whoever last spoke" to "a face in
> the asking room that claims a camera".

### ~~5. Turn ownership~~ — *superseded by item 9*

> "Which face is she talking to" turned out to be the wrong question. **Which *room* is she
> attending** is the right one: it survives a room having two faces in it, which is the actual
> arrangement on Android — a native client that owns the microphone and a WebView panel that
> draws her page. See item 9.

### ~~6. Echo, which the network makes worse~~ *(done 09/02/2026, v0.28.0)*

`Mute()`/`Unmute()` around her speech is what stops her transcribing herself, and it works
because everything is in one process on one clock. A tablet with an open mic and a speaker
playing her voice, both across a network with latency in each direction, will hear her and
transcribe it.

**Push-to-talk on the tablet for the first version.** It is honest, it is robust, and it
sidesteps the whole problem. Always-on listening there needs real echo cancellation — Android's
`AcousticEchoCanceler` is per-device and not dependable — and should be its own piece of work.

> **It was its own piece of work, and it is done.** Specced as *Stage 14 — Always-On Listening
> In A Room* and built in v0.28.0: `_openFace`/`_openRoom` as a quieter claim than the floor,
> a press that works on top of an open stream and hands back afterwards, and one room at a
> time. Measured at 74 seconds of her own voice into an open microphone producing no utterance
> at all. **This was the last item of Stage 14.**
>
> **The answer landed on the client, and that is the part worth keeping.** The host knows when
> it *sent* her voice; it does not know when a handset's speaker emitted it, or stopped. The
> client knows both exactly, because it owns the track — so the suppression lives there and
> nothing about it crosses the socket.
>
> That is Stage 15 item 3's rule arrived at from the other end, by somebody solving a
> different problem: **a device is best reasoned about by whoever holds it.** The owner stated
> it as a principle on the same day this was built as a necessity.

### ~~7. The attention gate now has two rooms~~ — *absorbed into item 9*

> Scoped rather than built out, which is what the spec asked for: one `AttentionGate` per
> room, constructed with the room, no shared statics. It only bites when a room gets
> always-on listening, which on Android it does not have and is not getting here —
> push-to-talk bypasses the gate entirely, because a held button has already answered the
> question it asks.
>
> **That last clause stopped being true on 09/02/2026, and the scoping is what saved it.**
> Item 6 gave a room always-on listening, so a phone's utterances now go through
> `Consider()` and are judged by **that room's** gate. The per-room gate went from a
> precaution to load-bearing in one release, and needed no work to get there — `EarsRoom`
> already resolved an open room, and the `_owed` fix carries the speaker across the gap
> between releasing and transcribing. *Scoping a singleton before you need to is cheap;
> discovering you needed to is not.*

### 8. `PROTOCOL.md` had fallen behind the code — ✅ **fixed in 0.20.1**

`setStats`, `setMicrophone`, `setOutput`, `setCameraDevice` and `setWhisperCompute` were all
handled in `OctaviaSession` and none were in the face→host table. `hello` was worse: eight
fields undocumented — `cameraDevice`, `stats`, `microphones[]`, `microphone`, `outputs[]`,
`output`, `whisperCompute`, `toolServers[]`.

Both are now written down. This mattered more than a tidy-up: an Android client is being
built against this document by someone who cannot see `OctaviaSession`, and a contract that
is quietly incomplete is worse than one that is honestly small.

### ~~9. Two rooms~~ *(done 09/01/2026, v0.24.0)*

> **Landed**, from [[Stage 14 - Two Rooms]] — a specification written on the Android side, by
> the client that needed it, for whoever was working in her repo. It supersedes item 5,
> absorbs item 7, and struck item 4 as already done.
>
> **Two faults, kept apart on purpose.** The first was five lines and security-shaped: *no*
> `set*` case in `OctaviaSession` looked at `inbound.From`, so the mic button on a handset at
> the gym opened the microphone in an empty house. `PROTOCOL.md` was honest that `listen`
> toggles her own microphone and nothing enforced it. There is an authority table now — host
> only, room, being — checked on the sending face's room before the switch acts.
>
> The second was the architecture. One `Conversation`, one `_state`, one `_mood`, and
> `caption`/`turn`/`state`/`emotion` all sent with no target, so every face was a window onto
> one room. `Conversation` is lifted out of `IBrain` — there are N conversations and one of
> her, and a `ClaudeBrain` per room would duplicate the client and the key to hold a list of
> strings apart.
>
> **Her voice was the one that was actively wrong** rather than merely coarse: `SendAudio`
> reached every face that had opted in, so she answered a question asked on a phone out loud
> at an empty desk. It takes a room now, and this machine's speakers are silenced for the
> length of a turn she is having somewhere else — the visemes and the streamed PCM untouched,
> only the sound card cut.
>
> **`_lastSpokenThrough` is gone, as its own comment asked.** `look` goes to a face that
> claims a `camera` in `ready`, in the room that asked. An absent `senses` is not an empty one
> — a face that predates the field is still asked, which is what keeps `attach-face.ps1`, the
> checks and the built-in page working untouched.
>
> **Rooms are serialised and stay that way.** One `_responding` flag, and the other room is
> refused out loud. Making it re-entrant is the concurrency change this defers: two rooms at
> once means two Whispers and two synthesis pipelines on eight cores, and a being holding two
> conversations is a worse simulation rather than a better one.
>
> All ten acceptance criteria are asserted in `EarsTest -- rooms`, in-process, against a
> recording transport and a stub local model — and each of the four mechanisms was broken on
> purpose first to watch the right checks go red.
>
> **All ten are also closed on real hardware**, from the Android side against this build.
> Criterion 7 was the one held open here, on the inherited belief that there was no API key;
> that belief was false and the round trip walked immediately on `--profile cloud`. A probe
> face joined room `phone` declaring `senses: []`, leaving the native client as the only
> camera in the room — so the WebView panel was never asked, which is exactly what `senses`
> was added for. Host and handset agree to the byte at 97 KB, and `setCamera` stayed per-room
> throughout with one `warn` line naming the room. See v0.24.1 in [[Changelog]].

### ~~10. Lending a renderer the device's senses~~ *(done 09/01/2026, v0.25.0)*

> **Landed**, from [[Stage 14 - Lending A Renderer The Device's Senses]] — the third spec
> written on the Android side and the first one produced by *this* side's own work: item 9's
> authority table and 0.24.1's secure-context fix were both right, and between them they left
> a handset offered neither a microphone nor a camera on a device that has both.
>
> **Neither was fixable on the wire**, which is what makes the seam interesting. The floor is
> a `FaceId`, so the WebView panel cannot press while the native connection streams; and
> watching is renderer-local by design and should stay there. So the page looks for
> `window.OctaviaEmbedder` and borrows — five changes in `bridge.js`, no protocol change, and
> a browser tab behaves exactly as it did.
>
> **A borrowed camera is not claimed to the host.** `senses` still reports what *this page*
> can do, because the embedder lends gaze rather than stills; claiming one would send `look`
> to a panel that cannot answer and, on Android, take that frame from the native client that
> can. There is a check that goes red if anyone tidies it.
>
> **The one honest difference:** the microphone button is press-and-hold on a room face and a
> toggle on the host. That is item 6 showing through — always-on listening in a remote room
> needs real echo cancellation — and it is stated in the code rather than papered over.
>
> Twenty-one assertions in `EarsTest -- embedder`, driving the real page in WebView2 across
> three faces, with the two origins reproduced rather than simulated. Four mechanisms broken
> on purpose first.

### ~~11. The first press of a cold session loses its opening words~~ — **closed 09/02/2026, v0.26.1**

> **Closed by removing the window rather than by buffering into it.** `OpenEarsOnStart`
> loads the speech models when the server starts, where nobody is talking — so by the time
> anyone holds a button the recogniser exists, `TakeFloorAsync` never awaits, and there is no
> gap for frames to fall into. Verified: a `talking` on a fresh session takes the floor with
> no *"her ears were shut; opening them"* line in the log, and `hello` reports
> `ears Whisper large-v3-turbo (local)` on the very first connection.
>
> Both fixes recorded below — buffering the pressing face's frames, or acknowledging
> `talking` — would have made the loss *survivable*. This makes it not happen. The residual
> window is the first two seconds of the server's life, which nobody is speaking into.
>
> **It came from the owner asking for something else entirely** — *"when the server starts up,
> the ears should auto start, no point in that only being activated when its needed"* — which
> is the second time a convenience request has turned out to be the clean fix for a logged
> fault. Worth noticing as a pattern: *a fault that needs a mechanism to survive it is often a
> fault that should not be reachable.*

*Reported 09/01/2026 from the handset, against v0.25.1. Fixed 09/02/2026.*

v0.25.1 made holding the button open her ears, which is right and confirmed working from the
phone — `micAccepted: True` on a fresh session, and the placard went from `EARS not started`
to `EARS Whisper large-v3-turbo` without anybody going near the desk. But opening Whisper
takes time — **measured at about 3 seconds with `large-v3-turbo` on CPU** — and the audio
streamed during it is thrown away:

```csharp
_face.AudioReceived += (from, pcm) =>
{
    if (_floor == from) _faceMic?.Push(pcm, pcm.Length);   // OctaviaSession.cs:121
};
```

The client opens its microphone and starts streaming the instant it sends `talking(true)`,
because **there is no acknowledgement that the floor was granted** — so every frame that
arrives before `_floor = from` is dropped, silently.

> **This is worse than the failure it sits next to, and that is the whole reason it matters.**
> Letting go while the model loads yields *silence*, which is legible: nothing happened, and
> you press again. This one yields a **plausible, truncated sentence** — press, start
> speaking immediately, and the opening words are gone with nothing to say so. She answers
> the wrong question and everything appears to work.

It cannot be fixed from the client, which has no signal to wait for. Two host-side shapes:

- **Buffer frames from `_pressing`** and hand them to `_faceMic` when it opens. No protocol
  change, no new message, and the client needs no edit at all. Needs a bound — a stuck press
  must not grow it without limit; 16 kHz mono 16-bit is ~32 KB/s, so a few seconds is cheap
  and a cap is one line.
- **Acknowledge `talking`**, so the client streams only once the floor is granted. Correct,
  and costs a message and a round trip on every press.

The first looks right: it is smaller, it is invisible to every existing client, and the
latency it hides is real rather than added.

**Only the first press of a session is affected.** Once the recogniser exists,
`TakeFloorAsync` reaches `_floor = from` without ever awaiting, so a warm press takes the
floor on the calling thread and there is no window at all.

### Order

1 → 3 → 2 → 6 → 4 → 5 → 7, with 8 folded in as the document is touched. Audio *out* before
audio *in*: it is the smaller change, it is independently useful, and it makes a tablet worth
looking at before it is worth talking to.

**How it actually went:** 1 → 3 → 2 → 9 → 10 → 11 → 6, with 9 taking 4, 5 and 7 with it and
10 falling out of 9. **Stage 14 is complete**, closed 09/02/2026 by the item everybody
expected to defer indefinitely.

> **And it made Stage 15 item 3 cheaper rather than dearer.** Yesterday's note here said the
> opposite: that once the server held no device and the Windows client streamed like a phone,
> the desktop would inherit the phone's echo problem, so item 6 became a prerequisite. That
> reasoning was sound and the conclusion is now moot — **the echo answer already lives on the
> client**, because only the client knows when its own speaker emitted. Item 3 inherits a
> solved problem instead of creating one.
>
> The phone reached that by necessity and the owner stated it as a principle the same day.
> When a constraint and an implementation arrive at the same shape independently, the shape
> is probably right.

## Stage 15 — A server, and clients *(specced and built 09/02/2026, v0.26.0)*

> **Item 1 is done.** Specced first, in the vault as *Stage 15 — A server, and clients*, and
> built in one change set because most of it turned out to already exist. `Octavia.Core` is
> her, `Octavia.Server.exe` is a headless host, `Octavia.exe` is a window that contains none
> of her. **Nothing on the wire changed**, which is the bill coming due on Stage 3's claim:
> the Android client connects to the new server unmodified.

The ask: *"the AI and all the actual work gets moved to a different server `.exe`… and then
have a client version on windows like how she is now just connect onto that instance, the
same with the android app."*

**Three pieces of evidence made this small.** The session had three references to a UI
framework in 9,361 lines and only one inside itself; `FaceHub` already took a nullable page,
so a server was an argument rather than a refactor; and `RoomChecks` had been running her
headless for a version already, which makes that harness a server minus a listener.

### 1. The split — **done**

`saveDiagnostics` lost its file dialog (a server has no dispatcher and nobody looking at it);
the page lost its `postMessage` transport and gained reconnection with a persistent bar; and
`Hotkey`/`StartMinimised` moved to `client.json`, carried over on first run. `EarsTest --
split` guards the boundary as text, because a compiler cannot — all three assemblies
legitimately see each other's internals.

### 2. A portable core — **open**

`Octavia.Core` still targets `net10.0-windows` with WPF on, for exactly one method:
`Sight.Inspect` greys a camera still with `BitmapFrame`. It works headless — WIC underneath,
no dispatcher — so it costs nothing today. Everything else is already behind an interface:
Whisper.net and ONNX are cross-platform, and NAudio and `System.Speech` sit behind
`IAudioSource`, `ISpeechRecognizer` and `IVoice`.

**So a Linux server is one image decoder and one `Octavia.Windows` project away**, and that is
the prize worth naming: an always-on box in a cupboard wants to be Linux.

### 3. The server holds no device — **decided 09/02/2026, open**

> **The owner's rule, and it settles this rather than opening it:**
>
> *"The server will always be the most powerful PC I have at the time. The client should
> always be the one passing the devices to the server. The server should have no hook on any
> device."*
>
> *"The phone sends its mic/camera/etc to the server — the Windows client should be doing the
> same thing."*

So the answer is not (a) or (b) from the earlier draft. It is **the phone's design becomes
the only design**, and the desktop's privileges are *deleted* rather than generalised.

That is a simplification, and the tell is that it needs no new concept: `talking`, `sight`
and the floor are already how a face lends a device. **The desktop stops being a special case
and becomes a face that happens to be on the same box.** Whether it is or not becomes a
deployment coincidence rather than something the code believes.

It also resolves the tension the split created. The server wants the strong machine because
Whisper, the local model and the voice all live there; a MetaHuman renderer wants a strong
GPU and is a *client*. In practice the same box runs both — and under this rule that costs
nothing, because nothing in the code assumes it.

**What moves out of the server:** `LocalMicSource`, `MicLevelMeter`, `AudioDevices`,
`LoopbackListener` and `MusicWatcher`, and playback of her voice. What stays is compute and
state: the brain, Whisper, the gate, the rooms, the tools, the config.

Four consequences that make this bigger than it looks, and each should be costed before
starting:

- **`music` has to travel upstream, and that is the first protocol change since Stage 3.**
  "What this machine is playing" becomes "what the *client's* machine is playing", which is
  more correct for a companion in a room. Streaming loopback audio to the server would be
  absurd, so the client runs the beat detection locally and sends a `music` message *up* —
  which is the standing constraint working exactly as written: reflex-speed things stay local
  to the renderer.
- **`SapiVoice` has to change or go.** It synthesises *to a sound card*, so a server with no
  device hook has nothing for it to do. It can be pointed at a stream instead — that is a
  supported thing SAPI does — but the current implementation cannot survive as written, and
  the neural voice becomes the default in practice rather than by preference.
- **The authority table shrinks and splits.** `setMicrophone` and `setOutput` become *client*
  settings — which device this client lends — and stop being host-only because there is no
  host device to protect. `listen` becomes "stream my microphone continuously", which is a
  room concern. Only `setWhisperCompute`, `openDataFolder` and `saveDiagnostics` stay
  server-side, because those really are about the machine she runs on.
- ~~**Item 6 stops being optional.** Always-on listening from a client with speakers in the same
  room is the echo problem, and today the desktop escapes it only because the server owns both
  the microphone and the mute. Once the desktop streams like a phone it inherits the phone's
  problem. It is more tractable than the general case — the server knows when it is speaking
  and the round trip on a LAN is milliseconds — but it is no longer somebody else's item.~~

  > **Struck 09/02/2026 — item 6 landed the same day and this consequence never arrived.** The
  > reasoning was right about where the problem would appear and wrong about who would own it.
  > v0.28.0 put the suppression **on the client**, because only the client knows when its own
  > speaker emitted and when it stopped. So the desktop does not inherit a problem when it
  > starts streaming like a phone; it inherits a phone's *answer*, already written and already
  > measured. This is the one bullet of the four that got cheaper instead of dearer.

**Until this is built: run the client on the server's machine, or use the neural voice.** Her
voice plays through the server's sound card for the host room, and SAPI cannot be streamed to
a client at all.

### ~~4. The server as a Windows Service~~ — **done 09/02/2026, v0.30.0**

> **Both halves.** The client half landed in v0.28.2; the service itself in v0.30.0, and the
> owner's correction is what kept it on the list: *"it may not always be the case"* that the
> server and the client share a box. They do today, and the item was still worth building.
>
> `--install`, `--uninstall`, `--start`, `--stop`, `--service-status`. Auto-start, so she is
> there after a reboot with nothing double-clicked. **Service mode is asked for explicitly
> with `--service` rather than sniffed from `Environment.UserInteractive`** — a wrong guess
> produces a process that neither runs as a service nor prints anything, and the flag is
> written into the registered command line once, where `sc qc` can read it back.
>
> **Starting and stopping her needs no administrator, which was the requested part.** A
> service is normally an administrator's object and every start would raise a UAC prompt —
> a poor thing to put between somebody and their own companion. `--install` splices one ACE
> into the service's own descriptor granting `RPWPDTLOCRRC` to the installing account, and
> nothing else: start, stop, pause, query. Not `WD` or `WO` — the right to hand *other*
> people control of her stays with administrators. **Start Octavia** and **Stop Octavia** are
> desktop shortcuts, and they simply work.
>
> **The clean shutdown finally happens from outside the process.** v0.28.2 recorded three
> mechanisms and two of them lying: `CloseMainWindow` skipped the unwind entirely, Ctrl+C was
> delivered and ignored, and Ctrl+Break could not be survived by whoever raised it. The SCM
> does properly what none of them could — `--stop` produced *"Octavia server stopped"* on the
> first attempt. **That is the argument for the service restated as a measurement**, and it
> is why the client still refuses to stop her itself.
>
> The client prefers the service: `LocalServer` asks `--start` before spawning a console,
> because a service outlives the window that wanted it and comes back after a reboot.
>
> **Two things found by building it.** The registered path was stored *unquoted* — invisible
> under `C:\Projects`, and the classic unquoted-service-path failure the moment anything sits
> under `C:\Program Files`, where Windows resolves `C:\Program.exe` instead. The cause was
> building the whole `sc` command as one string and letting two layers of parsing disagree
> about the quotes; `ArgumentList` fixed it and the registry value was read back to prove it.
>
> And **a service logs on as LocalSystem by default, so the hosted brain has no key**:
> `apikey.dat` is DPAPI-sealed to a *user account*. The local brain is unaffected, which is
> the profile she runs on anyway. `--install` says this at install time rather than leaving
> it to be discovered as a mysteriously broken brain.
>
> > **The fix was tried the same day, and it works** *(v0.30.1)*. The owner set the service
> > to log on as their own account, and everything that was in doubt was measured rather than
> > assumed: **her key decrypts** — she was asked a question on the `cloud` profile and Claude
> > answered — and *"her voice plays on this machine"* is in the log with no error behind it.
> > So DPAPI `CurrentUser` is readable by a service logged on as that user, session 0
> > notwithstanding, and the note above became *"here is the fix"* instead of *"here are two
> > things you might try"*.
> >
> > **Audio works in both directions, confirmed out loud by the owner** — *"she can still
> > hear me"* and *"I can hear her fine"*. That is the whole of the session-0 worry, closed
> > by the only method that could close it.
> >
> > It is worth stating plainly because the expectation ran the other way: **a service is in
> > session 0, and session-0 audio is the thing everybody warns you about.** Logged on as the
> > user it is simply full duplex — a microphone into Whisper, and her voice out of the sound
> > card, both audible. Enumerating a device proved nothing either way; a person listening
> > proved both.
> >
> > The cost is one that is worth writing down because it is silent: **a service logged on as
> > an account stops starting when that account's password changes**, and says so in the
> > Windows event log rather than in hers.
>
> Not done: a tray entry in the client for the same start and stop. The shortcuts are the
> answer to what was asked, and the tray is a nicety that can follow.

With the client starting it on demand, so double-clicking her still works. A console app
first, deliberately: a service that fails before its first log line is diagnosed by guesswork.

> **The second half is built and the first is still wanted.** Double-clicking her shortcut
> without the server up gave a thirty-second wait and a grey window that blamed a script
> error — the desktop shortcut written the day before, doing exactly what it was told. The
> client now starts her when she is local and absent, waits for the port, and only then
> loads a page.
>
> **The reconnection in `bridge.js` could never have covered this, and that is structural.**
> It recovers a socket that dropped; it cannot recover a page that was never served, because
> the retry lives in a file that has to be downloaded from the thing that is missing. *A
> server that goes away and a server that was never there are different faults.*
>
> It was argued here that this made the service unnecessary, because the server and the
> client always share a box. **The owner corrected that and was right** — *"it may not always
> be the case"* — and the correction is worth more than the claim: a deployment is the one
> thing in this project guaranteed to change, and item 3 exists precisely to stop the code
> believing in one. The service is still the item; this is the on-demand half of it.

**What stopping her taught, before the service does it properly.** Three mechanisms were
measured against a running server, and two of them lie:

| | |
|---|---|
| `CloseMainWindow` | Returns true, process gone in under 6 s, **handler never runs** — nothing logged, her sound card and any MCP child released by the OS. Closing the same console by hand logs correctly, so the two are not equivalent. |
| Ctrl+C | Delivered and **ignored**. `AttachConsole` and `GenerateConsoleCtrlEvent` both return true; she carries on serving. A server started by a window does not answer to one. |
| Ctrl+Break | **Works** — arrives as SIGQUIT, clean shutdown line every time. Also cannot be survived by whoever raises it: `SetConsoleCtrlHandler(NULL, TRUE)` ignores Ctrl+C only, and a real handler returning true did not save the caller either. |

So the client does not stop her at all. She is stopped by her own console's close button,
which is the SIGTERM path and is proven — and by the SCM once item 4 is built, where none of
the above applies. A client that took itself down to stop her would skip its own `OnExit` and
leave a dead tray icon: a tidy server for an untidy desktop.

### 5. Diagnostics bundles over HTTP — **open**

The path goes back over the socket now, which is enough when the client shares a machine with
the server and no help at all when it does not.

### The cost, stated before it was paid

- **Security stopped being optional.** `RemoteAccess` was opt-in and the desk worked without
  it; the socket is now the only way in, so the bearer key over plain HTTP is load-bearing —
  on a machine whose firewall is off entirely. That is the next thing to fix.
- **One process, one log, one breakpoint** was what made her pleasant to debug. *"She didn't
  answer"* now has a second place to hide.

## Standing constraints

- Anything reflex-speed (lip sync, dancing, level meters) is local; the model is for
  thought. The API bill should track conversation, not liveliness.
  **Since v0.26.0 this means local to the *renderer*, not in-process** — the viseme tap and
  the PCM still leave the same buffer at the same instant, and the socket carries both.
- The key never reaches a renderer. New capabilities land in the host.
- Every renderer change must leave `OctaviaSession` untouched; every sense change must
  leave the face untouched. If a stage wants to break that, the stage is mis-designed.
- Native runtimes that link CUDA stay out-of-process where practical. One process
  should not host two of them.
```
