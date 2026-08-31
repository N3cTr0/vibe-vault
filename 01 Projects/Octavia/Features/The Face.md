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

The full message list lives in [[Face Protocol]], which is regenerated from `PROTOCOL.md` on every vault sync. **It is not repeated here**, because the copy that used to be sat well behind the code and read "that is the entire contract" while being wrong about it — the same drift that cost the Android client a debugging round in v0.20.1. See [[The Host-Face Bridge]] for why the seam is shaped this way.

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

**But a face that never parsed never runs the code that reports anything**, and that hole stayed open until v0.19.3. A JavaScript syntax error is invisible to `dotnet build` and to the test harness, so the build goes green and the face is simply dead — which is how v0.18.0 shipped a broken bridge. Two things closed it:

- **`MainWindow.WatchForFace`** gives the page 30 seconds to send `ready` and otherwise logs an error and shows the fallback panel — a surface that does not depend on the renderer working, which is the entire point. The grace is deliberately generous because `ready` is sent when the socket opens, not when the scene finishes.
- **`EarsTest`'s `SyntaxChecks`** loads the real page in WebView2 over the same virtual origin and lets Chromium parse it, asserting no `SyntaxError`, that `window.Face` was published, and that the bridge would have sent `ready`. It was proved by breaking it on purpose: an orphan `});` in `bridge.js` names the file and line. Run it alone with `dotnet run --project tools\EarsTest -- syntax`.

Since v0.5.0 the page also carries the self-test results with their remedies, this machine's facts, and the recent log — a **Health drawer** then, and since the v0.10.0 rebuild the **Health tab** of the single drawer that replaced all three. See [[Diagnostics]].

## The chrome, as it settled

The console was rebuilt in v0.10.0 and refined through v0.19.x. What it is now:

- **One drawer, four tabs** — Transcript, Settings, Health, and Dev when offered — replacing three hand-written drawers with their own headers and close buttons. Its button sits top right, outboard of the state pill.
- **The status readout floats over the room** at the top left on translucent glass, and can be switched off entirely (*Settings → Show the status readout*) for when you want to look at her rather than her telemetry.
- **The caption is a subtitle, not a strip.** It sits *inside* the stage on a gradient scrim and fades after nine seconds of quiet. It has to be an overlay: as a sibling below the stage, hiding it changed the stage's height, fired the canvas `ResizeObserver` and re-framed the camera — she visibly jumped size twice a cycle. See [[The Room]].
- **Typing costs a click**, because most turns are spoken. The field opens focused and stays open until dismissed or a minute idle, and never discards a half-written line.
- **The API key is a setting, not a status.** It moved out of the strip into Settings, where it can nag the person who can act on it rather than the person looking at her.

## Fonts and assets

No CDN. three.js r180, `GLTFLoader`, `BufferGeometryUtils` and `@pixiv/three-vrm` are vendored into `wwwroot\lib` with their bare import specifiers rewritten to relative paths — so there is no import map and the CSP stays `script-src 'self'`. The prototype's Google Fonts were replaced with Segoe UI / Cascadia Mono. An always-on desktop app should not need the network to draw its own face.

## Next

Stage 4 retires the bust for a **VRM avatar** with 52 ARKit-style blendshapes, expression states, a headphones prop, and a shader background. The viseme→openness table generalises to viseme→blendshape-set. Stage 3 moves the transport to a WebSocket first, so the renderer becomes swappable rather than embedded. See [[Roadmap]].
