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

- **`MainWindow.WatchForFace`** gives the page 30 seconds and otherwise logs an error and shows the fallback panel — a surface that does not depend on the renderer working, which is the entire point. **Since v0.26.0 it means something narrower**: the client cannot see whether `ready` arrived, because `ready` goes to a server it has no channel to. What it can still see is whether *its own page* got as far as publishing `window.OctaviaFace`, which is the fault this was built to catch — a JavaScript parse error is invisible to `dotnet build` and leaves a dead face with a green log.
- **`EarsTest`'s `SyntaxChecks`** loads the real page in WebView2 and lets Chromium parse it, asserting no `SyntaxError`, that `window.Face` was published, and that the bridge would have sent `ready`. It was proved by breaking it on purpose: an orphan `});` in `bridge.js` names the file and line. Run it alone with `dotnet run --project tools\EarsTest -- syntax`.

> **It serves the page from a real socket now.** It used to load from a WebView2 virtual host and read `ready` off `postMessage` — but v0.26.0 deleted that channel, and `ready` is sent from the socket's `open` handler, so a page with nothing to connect to correctly says nothing. Loopback HTTP is a secure context, so the check gets the same CSP and the same `getUserMedia` availability it always had, *and* it now exercises the serving path as well. See [[A Server, And Clients]].

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

## The caption stopped covering her *(v0.50.1)*

> *"If there is a lot of text it takes over her entire window. Maybe have it scroll at the bottom where it already is, depending on where she is with saying it?"*

The placard was bottom-anchored with **nothing capping it**, so a six-sentence reply grew upward until it hid the one thing on screen worth looking at. It is a subtitle; it behaves like one now — three lines at the bottom, and the rest scrolls.

Three lines is `3.75em` against a `1.25` line-height, so it is three lines at every size the responsive clamp resolves to rather than a pixel count that happens to be right on one monitor. Whichever edge has text beyond it is softened. A caption short enough to fit has neither, and **does not take the mouse**: `pointer-events` go on only when it actually overflows, so a two-word reply never becomes a dead patch over her face.

`text-wrap: balance` went with it. It is for headlines and fights a scrolling box — it evens the last line by reflowing all of them, so every new sentence reshuffled the two above it.

![[v0.50.1 - three lines at the bottom, holding where she started.png]]

### And then it did follow her *(v0.51.0)*

The first attempt at *"depending on where she is with saying it"* followed the **writing** and was reverted the same evening. `OctaviaSession` re-captions after each sentence, which looks like a speech clock and is not one: `Say` hands a line to the engine and returns. A brain that outruns its voice put the last sentence on screen while she was still saying the first.

> *"It didn't follow her, I was watching — she was still saying the top stuff when it switched to the bottom."*

Three of the four pieces already existed. The [[The Voice|`Pacer`]] pulls audio at the sample rate, so paced samples is a **true** speech clock — it was built to be her clock when the sound card went away. The face receives audio paced and in order. `KokoroVoice` sees every frame through the tap. What nothing said was **where one utterance's audio ended and the next began**.

| the engine | writes `\x01end <n>` on stderr after each utterance — the running total of samples it has written to stdout |
|---|---|
| `KokoroVoice` | keeps two totals: received from stdout, and released by the `Pacer`. When the paced count passes a boundary, `Spoke(index)` fires |
| `OctaviaSession` | sends `sayingAt` with that sentence's **character range** in the caption text |
| the face | scrolls the range into view, one line of lead-in above it, and only when it is outside the box |

**The count travels with the marker** rather than the host counting stderr against stdout: those are two pipes with two buffers and their order relative to each other is not a promise. A number is a fact whenever it arrives.

**A range, not the sentence.** The face already has the words and is only being told which of them she has reached; two copies of a sentence are two things that can disagree.

**The cue is sent from both directions**, because either side can be late: a brain that outruns the voice composes a sentence long before its cue, and a brain that lags gets a cue for a sentence that does not exist yet. Whichever arrives second lands.

Not `scrollIntoView` — that scrolls every scrollable ancestor, so on a short reply it moves the whole page to chase a caption that was already visible.

![[v0.51.0 - mid-sentence, on the line she is actually saying.png]]

Eight sentences, composed in one burst at 22:20:45; the caption is on Mars and Venus with Mercury scrolled above and the next line faded. Watched in the browser too, which is the measurement that matters:

```
t=19.2  357 chars  scrollTop 0     ← the whole reply has arrived, the caption has not moved
t=23.5  357 chars  scrollTop 237   ← she is still speaking
t=25.5  357 chars  scrollTop 316
```

The text stopped growing at 19.2 seconds and the caption went on advancing for another six.

> **`EarsTest -- voice` pins the two-process contract** — three markers parsed, eight things that must not be mistaken for one. That seam is the likeliest in the project to rot silently: a marker that stopped being recognised would not throw, would not log and would not stop her speaking. The caption would go back to sitting still, which is what it did before any of this existed. See [[Lessons Learned]].

### Smoother than the browser's own *(same release)*

> *"Make it smoother, it was still jumping."*

`scrollTo({behavior:'smooth'})` moved eighty pixels in about three frames. Its duration is the browser's business and is tuned for a person clicking a link, where *getting there* is the point. Here the **travel** is the point — she is reading a line out loud and the caption should arrive about when she does.

So the tween is ours: `easeInOutCubic` over a `requestAnimationFrame` loop, scaled by distance and capped, so a short hop is not given the same second as a long one. A sentence-length move is now four hundred milliseconds across ten frames rather than sixty across one.

A hand on the wheel cancels it. Without that, reaching in to re-read a line is a tug of war with an animation that is still running, which feels like the caption fighting back.

### The room to herself *(same release)*

The status readout is **off by default** now. It was on, because it answers most of the questions somebody asks about her — true on the day she is set up, and false every day after, when the answers are the same as yesterday's and five lines of telemetry are lying across the room she is standing in.

What survived of it is a version and a word, bottom right at 28% opacity: `0.51.0 · local`. Those are the two facts that actually change.

**`local` or `cloud` is where the *thinking* happens**, not which model — cloud is the one worth saying out loud, because it means her side of the conversation is leaving the building. Re-read on every `hello`, since the profile can be switched under her and a stale `local` is worse than no word at all.

The setting still exists and still turns the panel back on. The default moved; the choice did not.

![[v0.51.0 - the room to herself, with a version in the corner.png]]
