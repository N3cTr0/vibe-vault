---
project: Octavia
tags: [octavia, spec, stage-14]
---

# Stage 14, item 10 — Lending a renderer the device's senses

> **Built, 09/01/2026, v0.25.0.** Added by the repo side above the original text, which is untouched below. Numbered 10, as suggested. All five page changes landed as specced, the trap is written into `PROTOCOL.md` rather than only into the client, and the press-and-hold difference is stated in the code rather than papered over. Acceptance 1–5 and 7 are asserted in `EarsTest -- embedder`, which drives the real page in WebView2 across three faces with the two origins reproduced rather than simulated; **acceptance 6 — a `look` arriving mid-watch — is the embedder's half and is owed from the client side.** See [[Lending A Renderer The Device's Senses]] and [[Changelog]] 0.25.0.

> **Item 10's host half is closed, and it found one more of item 9's** *(09/01/2026, v0.25.1)*. The restored microphone button could not have worked: `listen` was starting the **recogniser** as well as this machine's microphone, and item 9 had locked both — so a room face could never open her ears, and `micAccepted` reported "already started" rather than "will accept". Both fixed; holding the button now opens them. Reported from the handset against v0.25.0 with `micAccepted: False, ears: not started` on a live socket, which is the kind of evidence that turns a guess into a fix. See [[Changelog]] 0.25.1.

> **A specification, not a change.** Written 09/01/2026 from the [[Octavia Android]] side, which is the consumer that needs this, for whoever is working in her repo. Nothing in her repo was touched to produce it. It follows [[Stage 14 - Two Rooms]], which closed item 9 — number this whatever suits.

## The ask, in the owner's words

> I want us to somehow put back the microphone button on the phone and wire it in correctly. I don't like the key button press — **the UI should feel the same on the phone and the host**. Also, if I switch on the camera will her face and eyes follow me?

Two requests, and they turn out to be one seam.

## Where this stands after 0.24.1

Her page now hides two controls on a face with `controls: "room"`, and both for good reasons:

- **The microphone**, because it sends `listen`, which toggles the *host machine's* microphone. Item 9 made the host refuse that from another room.
- **The watch button**, because `navigator.mediaDevices` does not exist outside a secure context, so on a plain `http://<lan-ip>` origin it could only throw.

Both are correct, and both leave a remote face **less capable than the machine it is standing in for** — on a handset that *has* a microphone and *has* a camera, and whose native client already owns both. The phone's way in is currently a hardware volume key, which works and is not what anybody wants to look at.

## Why neither can be fixed on the wire

**The microphone.** `talking` takes the floor, and the floor is a `FaceId`:

```csharp
private FaceId? _floor;
...
if (_floor == from) _faceMic?.Push(pcm, pcm.Length);   // OctaviaSession.cs:122
```

The face that presses must be the face that streams. The Android app is **two** faces — a native connection that owns the microphone and a WebView panel that draws her page — so a button in the panel taking the floor would have the host dropping the native client's audio as coming from someone who does not hold it. Making the floor room-scoped instead would mean any face in a room could feed her ears, which is a worse rule than the one there is.

**The camera.** Watching is renderer-local by design, and `PROTOCOL.md` is emphatic:

> Watching — her following you with her eyes — is not in this protocol on purpose. It is a renderer-local mode: a person presses the face's camera button, a motion centroid is computed inside the page, the gaze moves, and nothing — no frame, no coordinate, no flag — crosses the socket.

That is the right design and this spec does not touch it. The problem is only that "the page" is the wrong place to *get the pixels* when the page is embedded in something that has a camera and the page's own origin is not secure.

## The seam

**A renderer may be embedded in something that has senses, and can borrow them.**

The page looks for an embedder. If one is there, a room face gets its controls back; if not, nothing changes and the buttons stay hidden exactly as they are today.

```js
/* Injected by whatever is hosting this renderer, if anything is. A browser tab has no
   embedder and never will; an Android WebView, an iOS one, or an Unreal shell can each
   provide one. Absent is the normal case and must stay the quiet one. */
window.OctaviaEmbedder = {
  /** Take or release the floor, using the embedder's microphone. It owns the socket
   *  that streams, so the page never sees a sample. Mirrors `talking`. */
  talking(held) {},

  /** Start or stop local gaze tracking with the embedder's camera. The embedder calls
   *  `window.Face.look(x, y)` itself, at whatever rate it likes, and `look(null)` when
   *  it stops. Nothing about it reaches the host — the same promise `watch.js` makes. */
  watch(on) {},

  /** What this embedder can actually lend. Read once, on load. */
  senses: ['mic', 'camera'],
};
```

**It is not an Android interface.** It is "the thing this renderer is running inside". That matters because the alternative — the page special-casing one client — is how a renderer stops being a renderer.

### What changes in her page

Small, and all of it in `bridge.js`:

1. `const embedder = window.OctaviaEmbedder ?? null;`
2. The microphone button is shown on a room face when `embedder?.senses.includes('mic')`. On `controls: "host"` it behaves exactly as it does now — `send({type:'listen'})`, untouched.
3. On a room face the button is **press-and-hold**, calling `embedder.talking(true)` on `pointerdown` and `talking(false)` on `pointerup`/`pointercancel`/`blur` — and the release must fire however the press ends, which is the same rule the Android client already follows for its key.
4. `watchBtn.hidden = !msg.camera || !(canOpenACamera || embedder?.senses.includes('camera'))`, and `toggleWatch` prefers `embedder.watch(true)` over importing `watch.js` when there is one.
5. The `watching` marker and the `aria-pressed` state are driven exactly as they are now. **The embedder must not draw its own privacy marker instead** — one marker, in the page, in the place a person already looks.

### The one place the UI cannot match, and it should be said out loud

The host's microphone button is a **toggle**: `listen` opens her microphone and leaves it open, and the attention gate decides what was addressed to her. A remote room cannot have that yet — always-on listening across a network with a speaker in the same room is the echo problem that item 6 deferred, and Android's `AcousticEchoCanceler` is per-device and not dependable.

So: **same button, same place, same look, held rather than toggled.** That is honest and it is the whole of the difference. A toggle would need real echo cancellation and item 7's per-room attention gate actually built, and both should be their own decision rather than smuggled in here.

If that difference is unacceptable, the alternative is not a smarter button — it is doing item 6 properly.

## The trap

**A still and a watch want the same camera.** `look` arrives from the host at any moment, and on Android `CameraStill` calls `provider.unbindAll()` — which would kill a running watcher mid-gaze and leave her staring at the last place she saw somebody. The embedder has to pause watching, take the still, and resume, and `Face.look(null)` in between so she is not frozen mid-glance while the shutter goes.

Worth stating in the protocol notes rather than only in the client: **any renderer that watches and answers `look` has this problem**, and a page-based one has it too — `watch.js` holds its own `getUserMedia` stream while `camera.js` opens a second one.

## What this does not change

- No new protocol messages, no new `hello` fields. The wire is untouched.
- The host still learns nothing about watching, and must not.
- `listen` remains host-room-only. The microphone button on a room face does **not** send it.
- A face with no embedder behaves exactly as it does today.

## Acceptance

1. On the handset, the microphone button is **visible** again and holding it makes her hear you — verified by a transcript, not by the button lighting up.
2. Releasing it ends the utterance, and so does dragging off it, backgrounding the app, or losing the socket.
3. The desktop's microphone button is unchanged and still toggles `listen`.
4. A plain browser on the LAN — no embedder — still hides both controls, and nothing throws.
5. Pressing watch on the handset makes her eyes follow a person moving in front of it, and the page's `watching` marker is up for exactly as long as the camera is.
6. A `look` arriving while watching returns a real frame, and her gaze resumes afterwards rather than sticking.
7. Nothing about watching appears in `octavia.log`.

## What the phone will do

Tracked on the Android side, listed here so the contract is visible from both ends:

- Inject the embedder over an **origin-restricted** channel — `WebViewCompat.addWebMessageListener` with an allow-list, not `addJavascriptInterface`, which exposes the object to every script in the WebView. The page arrives over plain HTTP on the LAN, so anything that can inject script into it could otherwise open the microphone.
- Map `talking` onto the same `startTalking`/`stopTalking` the volume key already drives, on the **native** socket.
- Implement `watch` with CameraX `ImageAnalysis`: the Y plane of `YUV_420_888` is already the greyscale `watch.js` computes by hand, so the port is smaller than the original. Same 64×36 grid, same thresholds, same smoothing and mirroring, ~8 Hz, pushing `window.Face.look(x, y)` back in through `evaluateJavascript`.
- Keep the volume key as well. It costs nothing, and it works with the screen off and with gloves on.

## Links

- [[Stage 14 - Two Rooms]] — item 9, which produced the hidden buttons this restores
- [[Stage 14 - A Microphone Somewhere Else]] — item 2, the floor and the push-to-talk rule
- [[Octavia Android]] — the client this is for
