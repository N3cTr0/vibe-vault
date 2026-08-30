---
project: Octavia
tags: [octavia, feature]
---

# The Room

*Stage 5, v0.6.0.* The wall she stands against, and the light on her.

## Why lighting and backdrop are one module

A warm evening key over a midday wall looks wrong instantly, and a room that dims without its lights dimming looks like a bug. So `environment.js` owns both, and moves them together.

Seven keyframes through the day — midnight, dawn, morning, midday, evening, dusk, midnight — each carrying a wall gradient, a halo colour, and the colour and intensity of the key, rim and ambient lights. The two bracketing frames are blended, so the room is never quite the same twice and never jumps.

The clock is re-read every ten seconds, not every frame: it takes an hour to change.

**It stays a room.** An early version let the wall go sky-blue at noon, and she immediately read as standing outdoors. The hours move the wall's *temperature and brightness*, not its species — it is plaster all day.

## What is in the shader

- A vertical gradient between the hour's two wall colours
- A **halo** behind her that grows with the microphone level, so the room reacts before she does
- Two very slow sine bands, so the wall is never perfectly flat
- A vignette
- Grain, matching the paper texture the page overlays on everything else

## Two things that had to be got right

**The backdrop is a fullscreen quad, not a plane in the scene.** A plane always finds a window shape where its edges come into frame, and sizing it generously just moves the gradient and vignette off-screen — which looks exactly like having neither. A quad drawn before the room has no edges, needs no aspect guessing, and skips perspective distortion. The main render then must not clear over it (`renderer.autoClear = false`).

**`smoothstep` with reversed edges is undefined in GLSL**, and the symptom is silence: it compiles, it runs, and the vignette simply is not there. See [[Lessons Learned]].

## Parallax

Two soft slabs drift between the wall and her at different rates — the cheapest convincing depth there is, and it survives any avatar.

They have **no edge at all**: a rectangle with a visible border reads as a mistake rather than a shadow, so their alpha falls off on both axes in a shader. The first attempt used flat-opacity planes and looked like two grey boxes.

## Looking at another hour

```js
Face.setHour(21)     // pin it
Face.setHour(null)   // follow the real clock again
```

Which is how the whole cycle was checked without waiting a day for it.

## Cost

A shader and three planes. That matters: the same GPU has a photoreal renderer, a speech model and Audio2Face in its future — see [[Roadmap]]. Anything reflex-speed here is local and cheap by design, exactly like the lip sync in [[The Voice]].

## Next

Stage 7 feeds the same halo from `music { bpm, energy, beat }` instead of microphone level, so the room pulses with what is playing. Nothing in this module has to change shape for that — it already takes a number and answers it.
