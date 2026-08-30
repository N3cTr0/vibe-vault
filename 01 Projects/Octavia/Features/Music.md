---
project: Octavia
tags: [octavia, feature]
---

# Music

*Stage 7, v0.8.0.* She hears what the machine is playing and moves to it, without calling a model.

## Why it needed a desktop app

This is the capability the whole architecture was chosen for. A browser page cannot hear the system — no web API exposes the output mix — so a companion that reacts to what you are playing has to be a process on the machine. Everything else in the host has a browser workaround; this one does not. See [[Architecture]].

## The chain

```
WasapiRecorder (loopback on the render endpoint)
  → LoopbackListener   downmix to mono, honour the Silent flag
  → MusicAnalyzer      onset envelope → tempo → phase
  → MusicWatcher       the decision, and what the face is worth telling
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

## The limitation, which is the machine's

> **This should disappear on the physical PC.** Everything below is a description of Remote Desktop's audio endpoint, not of her. Re-run `EarsTest -- music demo` early on the new machine and narrow the caveat to "over Remote Desktop" if the tempo settles. See [[Moving To The New Machine]].

**Remote Desktop's "Remote Audio" endpoint normalises everything to full scale** — crest factor 1.7, essentially a square wave — at any playback volume. A beat cannot be found in audio with no dynamics left in it.

This cost a debugging round and was worth the trouble to pin down, because every symptom pointed at the analyser. What settled it: delivery was 99.8% complete with one discontinuity, and playing at a *quarter* of the level returned byte-identical statistics. The analyser locks to within 0.3 bpm offline at every tempo and both common sample rates.

`EarsTest -- music` now prints the crest factor and says plainly when the path is limiting, so the next person is told rather than left doubting the arithmetic. See [[Lessons Learned]].

## Testing

| Command | Answers |
|---|---|
| `EarsTest -- beats` | The tempo, beat clock, speech rejection and hold behaviour. No audio device needed. |
| `EarsTest -- music` | What she makes of what is playing now, plus delivery and crest factor |
| `EarsTest -- music demo` | The same, but playing a known tempo through the speakers first — the whole chain, not the arithmetic in the middle of it |
