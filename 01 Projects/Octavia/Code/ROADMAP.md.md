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

## Stage 9 — Eyes and hands *(gate done 08/30/2026; eyes built, camera unverified)*

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

**Presence detection is not built.** It needs the camera, and the camera cannot be tested
here. It waits with it.

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
2. **Local-first profiles.** Agreed but not built: she should run mainly on the local
   model, with Claude added later for specific things. Concretely — base `Brain` default
   becomes `local`; the shipped profiles become `home` (local brain, good Whisper — the
   intended primary), `cloud` (Claude), and `dev` (local brain, `small.en`, for a weak
   machine); default `Profile` becomes `home`. Existing `config.json` files carry
   `dev`/`live`, so either migrate them or keep both names working.
3. **Docs and vault for the Stage 10 work** — `README.md` still describes the old
   console, `PROTOCOL.md` has no `camera` field note for the button, and there is no
   vault note for the rebuilt console. Screenshots for v0.10.0 have not been taken.
4. **Re-take the Stage 10 exit test** on the new machine, from an actual sofa.

---

## Standing constraints

- Anything reflex-speed (lip sync, dancing, level meters) is local; the model is for
  thought. The API bill should track conversation, not liveliness.
- The key never reaches a renderer. New capabilities land in the host.
- Every renderer change must leave `OctaviaSession` untouched; every sense change must
  leave the face untouched. If a stage wants to break that, the stage is mis-designed.
- Native runtimes that link CUDA stay out-of-process where practical. One process
  should not host two of them.
```
