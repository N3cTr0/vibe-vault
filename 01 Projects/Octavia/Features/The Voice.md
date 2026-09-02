---
project: Octavia
tags: [octavia, feature]
---

# The Voice

*v0.1.0 with Windows speech; the neural engine and audio-derived lip sync in v0.7.0 (Stage 6).*

## Two engines, one interface

`IVoice` joins `IBrain` and `ISpeechRecognizer` as a seam — see [[Architecture]]. `OctaviaSession` never learns which engine it has, and the engine can be swapped **under a running session**.

| Engine | Sounds like | Cost |
|---|---|---|
| `SapiVoice` — Windows speech | A 2010 satnav | Installed already, starts instantly |
| `NeuralVoice` — Piper | A person | ~80 MB once; 280–530 ms to first audio |

Windows speech is not deprecated scaffolding. It is on every machine, needs nothing downloaded, and starts instantly — the fallback that always works, exactly like the plaster bust in [[The Face]].

**She starts on Windows speech and upgrades herself** once the neural engine is ready, so a first run talks immediately rather than sitting mute through a download.

## Piper, out of process

Same reasoning as [[The Brain]]'s local model: a second ONNX runtime inside this process would sit beside Whisper's CUDA-linked one, and native dependency collisions are not worth the milliseconds saved.

So Piper is a **long-lived child process** — sentences written to its standard input, raw PCM read from its standard output, played through NAudio. Long-lived because the 60 MB model would otherwise reload for every sentence: cold start is ~2.5 s, warm is under half a second.

The engine and voices are fetched on first use into `<data>\voices`, like the Whisper models. **One difference worth stating: this downloads an executable**, not a model file. It happens only when the neural voice is asked for, it lands in her data folder rather than anywhere on the PATH, and the URL is in `PiperStore.cs` where it can be read.

`VoiceRate` maps to Piper's `--length_scale`, which is how long each phoneme is held — so the scale runs backwards from the one a person would expect.

## Lip sync, read out of the audio

**The plan assumed the engine would hand us phoneme timings. It does not** — and neither will most of what might replace it. SAPI gave visemes away for free; a neural engine gives a waveform and nothing else.

So the mouth is derived from the sound she is actually making:

- **Loudness → the jaw.** RMS against a decaying peak, so a quiet voice still opens her mouth and one loud moment does not leave her mumbling afterwards.
- **Spectral balance → the lips.** Energy in three formant-ish bands (250–900, 900–1800, 1800–3200 Hz). Their *balance* separates a spread "ee" from a rounded "ou"; the absolute level is loudness, which the jaw already has.

Two things had to be right for it to work at all:

- **Bins are weighted by frequency.** Speech tilts downward at roughly 6 dB per octave, so without compensation the low band wins every comparison and **every vowel reads as "oh"** — measured at 63% of voiced frames before the fix.
- **The front/back reference adapts.** Every engine and voice has its own spectral balance. Thresholds tuned on Piper made SAPI mumble in one shape. Tracking a running centre and expressing the boundaries as multiples of it gives a full spread on both: Piper 34/29/15/14/9%, SAPI 34/26/16/12/12%.

**It is analysed at playback, not at synthesis.** Piper runs ahead of the sound card once warm, so reading visemes off the generator would have her mouth finish a sentence a second before it was heard. `MouthTap` sits between the buffer and the output device and shows each block to the reader on its way past.

### Honest about what it is

It is not phoneme recognition and makes no claim to know which vowel she said. The shape boundaries were set from the measured distribution of real speech — a **distributional** choice, not a phonetic one: it guarantees her mouth moves through its whole range rather than settling into one shape, which is what reads as talking. Vowels carry most of a mouth's visible movement and are what those bands separate well; consonants it mostly gets out of the way for.

It also works for **any** engine we ever swap in, and Stage 7 needs exactly this DSP to make her dance — see [[Roadmap]].

`EarsTest -- mouth <file.wav>` prints the shape timeline and the front-axis quantiles. That probe is how the boundaries were chosen, and it is the only honest way to judge lip sync: it either looks like talking or it does not.

## Ending an utterance

Piper marks nothing. The end of speech is **quiet**: the playback buffer drained and nothing new arriving for 320 ms.

That produced the stage's best bug. Synthesis takes a moment to produce its first sample, and during that moment the buffer is empty and nothing has arrived — which is indistinguishable from having finished. She reached `idle` and *then* started talking. Nothing is over until it has begun, so the watchdog now waits for the first audio of an utterance before it is allowed to declare the end of one.

## The one that was pure comedy

A sound card is fed continuously, and `BufferedWaveProvider` returns silence when it has nothing. So the reader analysed that silence and sent a viseme **twelve times a second, forever** — every one of them saying "mouth shut". Visemes are now sent only when the mouth actually changes.

## Which sound card she comes out of *(v0.26.0)*

[[A Server, And Clients]] moved the being into its own process, and her voice went with it — so **she plays through the *server's* sound card**, for the host room, exactly as she always did. The Windows client hears her because it is standing in the same room, not because anything was streamed to the window.

That is fine while both halves are on one machine and it is the sharp edge of the split when they are not. A client in another building would leave the server talking aloud at an empty desk, and a **Windows voice cannot follow it**: SAPI cannot be streamed at all, so `audioAvailable` is false and a remote room is *told* — she can be read and not heard — rather than her falling back to speaking into an empty house.

> **So the neural voice stopped being an upgrade and became the way she talks**, for any arrangement where the client is not on the server's machine. The rest waits on Stage 15 item 3, which has the client lending the server its speakers; see [[One Being, Many Rooms]].

The exemption that went with it: the host used to refuse to stream audio to the page it hosted, because that page shared this machine's speakers and streaming would have been her talking over herself. There is no such page now.

## Still true from v0.1.0

The SAPI path still reports real viseme events, mapped to jaw openness *and* to a VRM mouth shape — see [[The Avatar Interface]]. And `Settle()` still clamps its queue counter after a `Hush`, because SAPI's cancellation events fire once per cancelled sentence and used to drive it negative.

## Where her voice comes out *(v0.35.0 — Stage 15 item 3)*

*Until now, out of the **server's** sound card. Now out of whichever face is listening — which on the desk is her own client.*

`speaker.js` plays the binary frames the host has been able to send since item 9, and the server sets `IVoice.Aloud = false` when any face in the attending room has subscribed to audio. On one machine that stops her being said twice, a fraction of a second apart.

> `turn in room 'host'; the face there is playing her, so this machine stays silent`

### `Aloud` already existed and already meant this

It was built for [[One Being, Many Rooms|item 9]], to keep the desk quiet while she answered a handset. Its own comment is the reason it fits: **"silencing the sound card rather than not speaking is deliberate"** — the visemes and the audio tee both read from the buffer *as it plays*, so cutting anywhere earlier takes them with it.

So this needed no new concept. One condition changed, from *which room is she attending* to *is somebody else playing her*.

### Subscribed only once it actually plays

The same rule the [[Lending A Renderer The Device's Senses|microphone fallback]] was rebuilt around, pointed the other way. An `AudioContext` that will not resume plays nothing, quietly and for ever — so a page that subscribed on the strength of *intending* to play would take the job from the sound card and then drop it in silence.

**No subscription means nobody claimed the job, and the sound card keeps it.** That is what makes this safe to switch on by default.

A browser refuses to start audio for an untouched page, so her own client passes `--autoplay-policy=no-user-gesture-required` — deliberate, because this window exists to be her and was opened on purpose. A plain tab retries on the first gesture instead.

### Hush had to be real

Frames already handed to the audio graph go on playing a sentence she was told to abandon, and nothing can un-schedule them after the fact. Every scheduled source is tracked and stopped on any state that is not `speaking`. The host has dropped its buffered audio on that same signal since item 9; this is that rule arriving where the sound actually is.

> **One visible consequence:** her voice now comes out of **`Octavia.exe`** in the Windows volume mixer rather than `Octavia.Server.exe`. Same speakers, different slider — worth knowing before concluding she has gone quiet.

### And then the sound card went entirely *(v0.36.0)*

*"We don't want the server having access to any local devices besides the GPU."* So the handover above stopped being a handover: there is nothing to hand back to.

**The hard part was never the loudspeaker.** `WaveOut` was the **clock**. It pulled from the buffer in real time, and three separate things hung off that pull:

| | |
|---|---|
| The audio teed to faces | Read as it went out, which is what keeps it in step |
| The visemes | Read from *the same bytes at the same instant* |
| The end of an utterance | Noticed by the buffer running dry |

One device doing three jobs, and only one of them made a noise. Delete it naively and her mouth stops moving; delete it carelessly and Piper's output races to a face seconds early.

`Pacer` replaces it — same buffer, same rate, same tap, no device. It is paced **against a stopwatch from a fixed start**, not by sleeping for the chunk length: every sleep is *at least* its duration and the work between is never free, so a loop drifts and a minute of speech arrives late with the visemes behind it. A machine that falls badly behind resets the clock to now rather than emitting a burst of catch-up audio.

`Aloud` is false permanently. Two earlier versions of that decision — *aloud in the host room*, then *unless a face is playing her* — were the same mistake at different sizes: both made the sound card the default and everything else the exception. **There is no default now.**

### One voice *(v0.37.0)*

`SapiVoice` is deleted. It had already stopped being a voice: it synthesises straight to a sound card, and its `AudioFormat` was **null** — the interface's own way of saying *this cannot be streamed to anybody*. Once the server lost its speakers, that made it not a lesser voice but **no voice at all**, speaking into a device that does not exist while every face waited in silence.

She starts on the neural voice directly. That used to be an upgrade she performed on herself — SAPI first, so a first run could talk while an 80 MB model downloaded, then a swap when it arrived. There is nothing to start with now, so **a first run is quiet until the model lands** and says so through `Trouble` rather than pretending. Honest silence beats a voice nobody can hear.

`setVoiceEngine: "windows"` is answered with a notice rather than obliged, and `VoiceEngine` still exists as a field because an existing `config.json` says `"windows"` and reading it has to mean something rather than crash. **A setting that can only be wrong is worse than no setting; one that silently does nothing is worse again.**

> **Seven checks were deleted rather than ported.** They asserted the mapping from Windows' 21 viseme ids onto the five VRM mouth shapes — a mapping that lived inside `SapiVoice`. The neural voice never sees a viseme id: `VisemeReader` derives her mouth from the waveform, which is exactly why it stays in step with audio that is *streamed* rather than played. A rewritten version would have asserted arithmetic nothing performs.

## The audition *(09/03/2026 — Stage 16)*

*"I need to search for a free or commercial voice for Octavia, the one she has now I don't like."*

Nothing here has changed yet. `tools/VoiceAudition` renders one paragraph in twenty-two candidate voices into `data\auditions`, because **which one is nicer is an ear, not a specification** — see [[Roadmap]] Stage 16.

The tool is free-standing on purpose: it does not reference `Octavia.Core`. sherpa-onnx carries its own native `onnxruntime.dll` and Octavia.Core carries Microsoft's for Silero and the wake word, so the challenger is auditioned in a process that has never met the incumbent.

### The two candidates

| | Piper, a different voice | Kokoro |
|---|---|---|
| Work to switch | one line of config | a new `IVoice`, roughly a day |
| Disk | 60–114 MB per voice | 350 MB once, every voice included |
| Streamable | yes, already proven | yes — chunked callback, raw PCM |
| Her mouth | unchanged | unchanged — PCM at 24 kHz, read the same way |
| Offline, private, free | yes | yes |

Ten Piper voices, including the three `high` models that were never on her shortlist — `high` is a larger model at the same sample rate, and the likeliest place for the incumbent engine to have a better answer hiding.

Twelve Kokoro voices, at 82M parameters against a Piper voice's ~20M. It clears every constraint Stage 16 wrote down *before* the search began, which is why it is the only new engine on the sheet. **Measured rather than assumed:** on this machine with no GPU it synthesised the audition paragraph at an RTF of **0.34–0.47** — about two and a half times faster than she says it, so "real-time, sentence by sentence" survives a model four times the size.

Hosted voices — ElevenLabs and the like — were left off deliberately: per-character billing on someone who talks all evening, network latency on every sentence, and an outage becoming muteness rather than plainness.

> **`kokoro-multi-lang-v1_1` is a trap.** Newer than `v1_0` and 103 speakers against 53, so it looks like the obvious download. One hundred of them are Chinese; it contributes three English women. The English catalogue is in **`v1_0`**. See [[Lessons Learned]].
