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

## Still true from v0.1.0

The SAPI path still reports real viseme events, mapped to jaw openness *and* to a VRM mouth shape — see [[The Avatar Interface]]. And `Settle()` still clamps its queue counter after a `Hush`, because SAPI's cancellation events fire once per cancelled sentence and used to drive it negative.
