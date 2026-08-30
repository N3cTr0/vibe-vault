---
project: Octavia
tags: [octavia, feature]
---

# The Face

*v0.1.0; restructured in v0.6.0.* A three.js renderer in a WebView2 — and, more importantly, one that knows nothing about anything.

## What it is

Four modules on three.js r180:

| File | Owns |
|---|---|
| `face.js` | The scene, the camera, the loop, and the *performance* — blinks, saccades, carriage, mood. Exposes `window.Face`. |
| `environment.js` | The wall and the light rig — see [[The Room]] |
| `bust.js` | The plaster bust, as an avatar |
| `vrm-avatar.js` | A VRM character, as the same avatar |

The last two implement one interface, which is what lets either take the same performance — see [[The Avatar Interface]].

**The bust** is a sphere with a real deforming jaw (a morph target built by rotating lower-front vertices around a pivot), brow bars, eyes with sweeping lids and independently aimed irises, a nose, a dark mouth aperture, neck, chest and plinth. Ported from the prototype essentially intact; it survived the move to a desktop app *because* it was already dumb, and it survives Stage 5 as the fallback that always works.

**Vendored libraries.** `three`, `GLTFLoader`, `BufferGeometryUtils` and `@pixiv/three-vrm` live in `wwwroot\lib` with their bare import specifiers rewritten to relative paths, so there is no import map and the CSP stays `script-src 'self'`. They are excluded from the vault's code snapshot: 3 MB of somebody else's code is not our source.

## What it does not do

No key. No model calls. Its CSP is `default-src 'none'`, and `connect-src` names only the loopback face socket and the read-only `https://octavia.avatar` origin the host maps — so it cannot reach the wider network, and it cannot even read a character file the host did not offer it. No audio. No decisions.

It receives `state`, `level`, `viseme`, `emotion`, `caption`, `turn`, `notice`, `hello`, `diagnostics`, and sends `ready`, `say`, `listen`, `hush`, `forget`, `setKey`, `setVoice`, `selfTest`, `saveDiagnostics`, `faceError`. That is the entire contract. See [[The Host-Face Bridge]].

It does not choose its own character either: the host names one in `hello`, on an origin the host maps.

## The rig

A single `tick()` loop drives everything from a small state object, and hands the result to whichever avatar is loaded:

- **Mouth** — `viseme` messages set the shape and weight; the avatar decides what that means. A jaw morph on the bust, five blendshapes on a VRM.
- **Mood** — `emotion` messages set an expression and weight, eased in rather than cut.
- **Blink** — randomised every 2–6 s, with a squint added while thinking.
- **Gaze** — saccades whose range and cadence differ per state: tight and steady while listening, wandering up and away while thinking.
- **Carriage** — head yaw is biased 0.115 rad to counter the three-quarter camera, so she appears to look *at you*. Listening adds a slight tilt; thinking turns away.
- **Breathing** — slow sine on position and pitch, disabled under `prefers-reduced-motion`.
- **Plinth arc** — a cobalt torus whose span tracks microphone amplitude while listening and whose spin rate tracks state.
- **Dance** — `music` messages bring the headphones down, put a nod on each beat and a sway across the bar, both scaled by energy so a quiet passage is a smaller movement rather than the same one switched off. The beat is an *impulse* the host sets and the loop decays; the face never runs a clock of its own. See [[Music]].

The level meter feeding that arc is why she reacts while you are *still talking*, before a single word has been transcribed.

Every one of those is reachable by hand from [[The Dev Panel]], which is the only practical way to look at a rare one.

## Diagnostics

The host has no browser console, so the face forwards `window.onerror` and unhandled rejections as `faceError` messages into `octavia.log` — now at `error` level, since a renderer that throws is not routine. The `ready` message carries `faceBuilt` — true only if `window.Face` exists, which means the WebGL context and the whole scene graph constructed without throwing. That one boolean is how the host verifies the renderer works headlessly, and it is what the Renderer line of the self-test reports.

Since v0.5.0 the page also carries the **Health drawer**: the self-test results with their remedies, this machine's facts, and the recent log. See [[Diagnostics]].

## Fonts and assets

No CDN. three.js r180, `GLTFLoader`, `BufferGeometryUtils` and `@pixiv/three-vrm` are vendored into `wwwroot\lib` with their bare import specifiers rewritten to relative paths — so there is no import map and the CSP stays `script-src 'self'`. The prototype's Google Fonts were replaced with Segoe UI / Cascadia Mono. An always-on desktop app should not need the network to draw its own face.

## Next

Stage 4 retires the bust for a **VRM avatar** with 52 ARKit-style blendshapes, expression states, a headphones prop, and a shader background. The viseme→openness table generalises to viseme→blendshape-set. Stage 3 moves the transport to a WebSocket first, so the renderer becomes swappable rather than embedded. See [[Roadmap]].
