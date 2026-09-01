---
project: Octavia
tags: [octavia, feature]
---

# Lending A Renderer The Device's Senses

*Stage 14 item 10, v0.25.0.* The page can borrow a microphone and a camera from whatever it is running inside.

Built from [[Stage 14 - Lending A Renderer The Device's Senses]], written on the [[Octavia Android]] side. It is the first of that series produced by **this** side's own work: [[One Being, Many Rooms]] hid two controls for good reasons, 0.24.1 hid a third for another good reason, and between them they left a handset holding a microphone and a camera and offered neither.

## The ask

> I want us to somehow put back the microphone button on the phone and wire it in correctly. I don't like the key button press — **the UI should feel the same on the phone and the host**. Also, if I switch on the camera will her face and eyes follow me?

Two requests. They turn out to be one seam.

## Why neither could be fixed on the wire

**The microphone.** The floor is a `FaceId`:

```csharp
if (_floor == from) _faceMic?.Push(pcm, pcm.Length);
```

The face that presses must be the face that streams. The Android app is **two faces** — a native connection that owns the microphone and a WebView panel that draws her page — so a button in the panel taking the floor would have the host dropping the native client's audio as coming from somebody who does not hold it.

Making the floor room-scoped instead was the obvious alternative and is worse: any face in a room could then feed her ears. The existing rule is one talker at a time, first press wins, and it is right.

**The camera.** Watching is renderer-local by design, and [[Face Protocol]] is emphatic that nothing about it crosses the socket. That is not the problem and is not touched. The problem is that *the page* is the wrong place to get the pixels when the page is embedded in something that has a camera and its own origin is not secure.

## The seam

**A renderer may be embedded in something that has senses, and can borrow them.**

```js
window.OctaviaEmbedder = {
  senses: ['mic', 'camera'],   // what this embedder can lend; read once, on load
  talking(held) {},            // take/release the floor on the embedder's own connection
  watch(on) {},                // start/stop gaze; it drives window.Face.look(x, y) itself
};
```

**Absent is the normal case and stays the quiet one.** A browser tab has no embedder and never will; the page behaves exactly as it did.

**It is not an Android interface.** It is "the thing this renderer is running inside" — an Android WebView, an iOS one, an Unreal shell. That matters because the alternative, the page special-casing one client, is how a renderer stops being a renderer. The same instinct as `IBrain` and `IVoice`: see [[Architecture]].

## Three decisions worth keeping

**A borrowed camera is not claimed to the host.** `senses` in `ready` still reports only what *this page* can do. It looks like an oversight and is the opposite: `senses` routes `look`, and the embedder lends **gaze, not stills**. A panel that claimed a camera would be sent a `look` it cannot answer — and on a handset would take that frame away from the native client, which can. There is a check that goes red if anyone tidies it.

**The privacy marker stays in the page.** One marker, in the place a person already looks. An embedder drawing its own would be two things to trust rather than one, and the whole promise of [[Eyes]] is that the marker cannot be bypassed.

**The embedder is injected over an origin-restricted channel.** The page arrives over plain HTTP on the LAN, so anything that could inject script into it could otherwise open the microphone. On Android that means `WebViewCompat.addWebMessageListener` with an allow-list, never `addJavascriptInterface`, which exposes the object to every script in the WebView. See [[Conventions & Security Model]].

## The one place the interface cannot match the host

The desktop's microphone button is a **toggle**. `listen` opens her microphone and leaves it open, and [[The Attention Gate]] decides what was addressed to her.

A remote room cannot have that yet. An open microphone beside a speaker playing her voice, across a network with latency each way, is the echo problem Stage 14 item 6 deferred — and Android's `AcousticEchoCanceler` is per-device and not dependable.

So: **same button, same place, same look, held rather than toggled.** That is the whole of the difference, and it is stated in the code rather than papered over. Making it a toggle needs real echo cancellation and item 7's per-room gate actually built. If the difference is unacceptable, the answer is not a smarter button — it is doing item 6 properly.

> Item 6 is now the **only** thing between the phone and the desktop feeling identical, which is a much sharper way to hold it than "echo, later".

## Every way a press can end

A held button that never releases holds her ears until the host's sixty-second floor timeout, which is the failure this cannot be allowed to have. `pointerup`, `pointerleave`, `pointercancel`, `blur` and `visibilitychange` all release, and the release is idempotent so the overlap costs nothing.

- **Dragging off cancels**, which is what a person expects of a push-to-talk button.
- **The system taking the gesture cancels** — on a phone that is a scroll starting, or a call arriving.
- **Backgrounding the app cancels.**

## A still and a watch want the same camera

Written into [[Face Protocol]] rather than left in one client, because **any** renderer that watches and answers `look` has it. `look` can arrive at any moment, including mid-gaze. A renderer that binds the device exclusively — Android's CameraX `unbindAll()` is the example — will kill the watcher and leave her staring at the last place she saw somebody.

Pause the watcher, take the still, resume, and set the gaze to nothing in between so she is not frozen mid-glance while the shutter goes. The built-in page has the mild version of this today: `watch.js` holds its own stream while `camera.js` opens a second one, which a desktop browser allows.

## What is proven

`EarsTest -- embedder` drives the **real page in the real engine** and injects the embedder with `AddScriptToExecuteOnDocumentCreated`, which is where a WebView host puts it. Twenty-one assertions across three faces: a handset with an embedder, a plain browser on the LAN with none, and the desktop in the host room.

**The two origins are reproduced, not simulated.** `https://octavia.face` is a secure context and `http://octavia.face` is not, so `getUserMedia` is genuinely absent on the second exactly as on a handset — and the first assertion checks *which one it got*, so a run that passes because the simulation broke says so instead of going quietly green.

Four mechanisms were broken on purpose to watch the right checks go red: the borrowed-microphone gate, the extra release paths, the `senses` claim, and the borrowed-camera watch gate.

> Two faults were in the harness rather than the page, and both are the kind that produce a green run for the wrong reason. `AddScriptToExecuteOnDocumentCreated` **accumulates** — without removing the previous script, the handset's embedder would still have been injected into the "plain browser" run, and the two checks proving a browser tab is left alone would have passed while testing the wrong page entirely. And `ExecuteScriptAsync` returns the completion value, which is not always a string.

## Links

- [[Stage 14 - Lending A Renderer The Device's Senses]] — the specification
- [[One Being, Many Rooms]] — item 9, whose guards created the gap this fills
- [[Eyes]] — watching, the marker, and the still/watch conflict
- [[The Ears]] — the floor, and why it is a face rather than a room
