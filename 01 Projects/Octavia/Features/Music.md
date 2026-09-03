---
project: Octavia
tags: [octavia, feature]
---

# Music

*Stage 7, v0.8.0.* She hears what the machine is playing and moves to it, without calling a model.

## Why it needed a desktop app

This is the capability the whole architecture was chosen for. A browser page cannot hear the system — no web API exposes the output mix — so a companion that reacts to what you are playing has to be a process on the machine. Everything else in the host has a browser workaround; this one does not. See [[Architecture]].

## The chain

> **Half of this chain no longer exists on the server** — v0.38.0, Stage 15 item 3, *"no local devices besides the GPU."* `LoopbackListener` and `MusicWatcher` were deleted with every other device class. **`music` now travels upstream**: a client hears what its own machine is playing and tells her. See [[A Server, And Clients]].
>
> **And nothing sends it yet**, so her music sense is dark on every face. The protocol carries it and the analysis still works; what is missing is a client that captures loopback, which a page structurally cannot do — it needs the WPF shell to push it through `window.OctaviaFace.send`. This is the oldest open item in the project.

As it was, and as the arithmetic still is:

```
WasapiRecorder (loopback on the render endpoint)     ← on a client now
  → downmix to mono, honour the Silent flag          ← was LoopbackListener
  → MusicAnalyzer      onset envelope → tempo → phase   ← still here, unchanged
  → the decision, and what the face is worth telling  ← was MusicWatcher
  → music { playing, bpm, energy } / music { beat }
```

`MusicAnalyzer` takes samples and returns numbers. **No device, no threads, no clock** — which is what lets its tempi be checked against generated tracks instead of by playing something and watching her.

## How a tempo is found

Standard chain, deliberately: a **spectral-flux onset envelope** (only rises count — a note ending is not an onset, and counting both turns every beat into two), **autocorrelated** across lags from 60 to 190 bpm, then matched against a **pulse train** at every offset within one period to find where the beat actually falls.

Three details earn their place:

- **A log-normal tempo prior centred on 120.** Tempo is perceived logarithmically, and without it the autocorrelation happily returns half or double what a person would tap — both are genuinely periods of the music.
- **Parabolic interpolation of the peak.** A tempo is almost never a whole number of hops; 150 bpm is exactly 37.5 of them. Correlating at the nearest integer understates the peak *unevenly*, so a bar line that happens to land on a whole number outscores the beat that does not.
- **An octave check.** Almost all popular music puts a different sound on two and four, which makes the bar more periodic than the beat. If twice as fast is nearly as periodic and still plausible, that is the answer. The threshold was **measured, not picked**: where doubling is right the half lag scores 0.63–0.67 of the winner; where it is wrong the two are anti-correlated at about −0.03. A wide gap with a sign to it.

## Music, not speech

Three things must agree, chosen because speech fails two of them:

| Test | Music | Speech |
|---|---|---|
| Loud enough to be something | yes | yes |
| **Continuity** — fraction of recent frames above the floor | high | low; speech is full of gaps |
| **Periodicity** at a steady tempo | high | weak and diffuse |

Continuity is the cheapest speech rejector there is that does not involve a classifier, and it is why the test material is a sustained bass and pad with transients on top rather than a bare click track. A click track would have passed a test that real music would fail.

## She keeps time while she talks

Her own voice comes out of the speakers and back through the loopback like anything else. Left alone she would hear herself and call it music.

The fix is not to stop listening but to **hold**: while she speaks, the analysis is frozen and the tempo already found keeps running from its own clock. So she stays in step with a track she is talking over, and nothing she says can start or stop the music behind her. Three checks in `tools\EarsTest` cover exactly this.

## What the face does

`setHeadphones` joins the avatar interface — the fourth thing an avatar must know how to do — and both the bust and a VRM implement it. The prop itself is one module sized to whatever head is asking, because the bust is a unit sphere and a VRM head is about fifteen centimetres.

The performance stays in the face, where blink and gaze live: a nod on the beat, a sway across the bar rather than per beat (per-beat reads as a twitch), and the room's halo answering energy with a ring that leaves on each pulse. Energy eases; the beat does not — smoothing it would turn every pulse into the same gentle swell.

**The face never runs its own beat clock.** It moves on `beat` messages. A face predicting its own beats from `bpm` would drift out of time within a few bars, and the host is the one actually listening. See [[The Host-Face Bridge]].

## What she is told about it

The brain gets one line saying music is playing and roughly how fast — attached to the **current question only**. Not the system prompt, which would void the cache breakpoint sitting there; not the conversation history, which would leave her insisting there is music an hour after it stopped. `IBrain.RespondAsync` grew an optional `context` for exactly this, and the camera will use the same seam. See [[The Brain]].

The line tells her not to mention it unless asked, because a model told "music is playing" will otherwise open every reply by saying so.

## Privacy

The loopback hears **everything** the machine plays, which could be a call as easily as a song. So:

- The analysis is local, and nothing is stored. Buffers are converted, measured and dropped; what survives is a tempo and a loudness.
- **No audio crosses the protocol.** There is no message that carries sound, and the diagnostics bundle contains none.
- **Settings → Hears what you play** turns it off, and off *closes the device* rather than ignoring it — a switch that left the capture running would be a worse promise than no switch.

## The limitation that was ours all along

> **Resolved 08/31/2026 in v0.12.0. It was never the machine.** The section below is kept
> because the reasoning is instructive and the conclusion was confidently wrong three
> times over.

**What was believed:** Remote Desktop's "Remote Audio" endpoint normalises everything to
full scale — crest factor 1.7, essentially a square wave — so no beat could be found. That
survived a debugging round: delivery was 99.8% complete with one discontinuity, playing at
a *quarter* of the level returned byte-identical statistics, and the analyser locked to
within 0.3 bpm offline at every tempo. Every symptom pointed away from the analyser, and
the endpoint was the obvious remaining suspect.

**What was actually true:** `LoopbackListener` was decoding the wrong bytes. A shared-mode
mix format is `WAVE_FORMAT_EXTENSIBLE`, not `IeeeFloat`, so the float test failed and the
decoder fell through to `ToInt16` — reading **the low two bytes of each 32-bit sample**.
Those bits are uniform noise. Uniform noise has an RMS of 0.577 and a crest factor of
1.73, and that is what was measured, to three decimal places, on *every* device.

**The tell was there the whole time and nobody looked at it.** The same 1.7 came back over
Remote Desktop on the VM, through a virtual streaming endpoint, through a Jabra headset and
through a Realtek card. Four unrelated devices do not share a defect; a decoder does. The
constant should have been read as evidence about our code rather than as three separate
confirmations of a theory.

**The other half of the mistake** was judging a captured crest factor against an absolute
threshold. The number means nothing without the crest factor of what was *played*. The
demo track's own is 7.7; after the fix the capture reads 7.7, with peak 0.793 and RMS 0.103
matching the source exactly, and the tempo settles at **131.8 bpm against a played 132 at
confidence 1.00** — where it used to wander between 75 and 184. `EarsTest music demo` now
prints both numbers and says "CAPTURE IS FAITHFUL" rather than accusing the machine.

Decoding now lives in `AudioSamples`, shared with the diagnostics, so a check cannot
disagree with the capture it is checking. See [[Lessons Learned]].

## Testing

| Command | Answers |
|---|---|
| `EarsTest -- beats` | The tempo, beat clock, speech rejection and hold behaviour. No audio device needed. |
| `EarsTest -- music` | What she makes of what is playing now, plus delivery and crest factor |
| `EarsTest -- music demo` | The same, but playing a known tempo through the speakers first — the whole chain, not the arithmetic in the middle of it |
