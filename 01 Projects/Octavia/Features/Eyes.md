---
project: Octavia
tags: [octavia, feature]
---

# Eyes

*Stage 9, v0.9.0.* One still, only when a question needs it. **Built, and honestly only half verified.**

## Who owns the camera

**The face owns the camera; the host owns the decision to open it.**

That split looks backwards against "the face is a renderer, the host is the being" — the face owns no audio device, so why a camera? Three reasons, and they were anticipated:

1. **`getUserMedia` needs a secure context.** The virtual `https://octavia.face` origin was set up in Stage 3 *knowing this stage was coming* — see [[The Host-Face Bridge]]. It costs nothing then and is awkward to retrofit.
2. **The camera belongs where the person is.** A face on a wall tablet has one pointed at the room; the machine under a desk may not.
3. The renderer never *decides* anything. It opens the device only on `look`, and `look` only comes from the host.

## The three gates, and why none of them is a model

Before a camera opens, all of these must be true:

1. `"Camera": true` in config
2. the words genuinely need eyes — `Sight.WantsEyes`, a plain word list
3. the brain has eyes at all — Claude; the local model does not

`WantsEyes` is a word list **on purpose**, and a model would be more accurate. Opening a camera in someone's home is the most intrusive thing she does, so the rule that triggers it has to be **legible**: a person should be able to read forty lines and know exactly what makes her look. A classifier would be better and nobody could audit it.

It errs towards *not* looking. A false negative is her saying she cannot see; a false positive is a camera turning on unasked. Those are not comparable mistakes, and the tests are written in that direction.

## Off by default — the only sense that is

The microphone is on when she is listening. The loopback is on by default. The camera is **not**, and that asymmetry is deliberate: a microphone in a room is expected, a camera is not.

The other promises, all enforced in code rather than documented and hoped for:

- **She never watches.** The device is opened, one frame taken, and the track stopped in the same breath — inside a `finally`, so a throw anywhere still closes it. There is no streaming mode to leave running.
- **It says so.** A red *camera on* bar covers the top of the window while the device is live. A camera that can turn on quietly is a camera nobody should agree to.
- **Nothing is stored.** The still rides with the one question that asked for it and never enters the conversation history — the same rule as the music context in [[The Brain]], for the same reason: a photograph from a quarter of an hour ago is not what she can see now.

## The seam it arrived through

Stage 7 added a loose `context` string to `IBrain.RespondAsync` and the vault predicted the camera would use "the same seam". It does, and the seam grew a name:

```csharp
record Situation(string? Context, string? Image)
```

Only the newest turn carries it. A brain that cannot use part of it ignores that part — `LocalBrain` has no eyes and simply drops the image.

## The host answers the permission, not the runtime

WebView2 raises `PermissionRequested` for anything a page asks for, and **if nobody handles it the runtime decides** — which quietly made `"Camera": false` a suggestion rather than a boundary, since the page could have asked and been granted over the host's head.

The host now answers every one of them from its own config: a camera request from her own origin with the setting on is allowed, and *everything else* — microphone, geolocation, notifications, clipboard, the lot — is denied outright. Nothing is saved in the browser profile, so turning the camera off in config takes effect at once rather than needing browser state cleared.

This was missing from the first cut of the stage and was only noticed when a real camera appeared to test against.

## Was that a picture, or a black rectangle?

**`Glance`** describes every captured frame — size, mean brightness, and *spread* — and logs it. The pixels are measured and dropped; nothing is stored.

It exists because a camera can open, report no error, and hand over nothing. A lens cap, a privacy shutter, a redirected device that never starts, and an unlit room all look **exactly like success** from the outside. This is the silent microphone of [[Diagnostics]] a second time, and it earned its place immediately: the first real capture came back at 0.15 brightness, which is how the exposure bug below was found at all.

Spread below 0.02 is reported as `BLANK, the camera opened but produced no picture`.

## What hardware taught it

The first version waited **two animation frames** — about 33 ms — before grabbing. That was plenty against a synthetic device and far too little against a real sensor, which needs a few hundred milliseconds for auto-exposure and white balance to settle. Now 450 ms.

Measured on the same scene, before and after: brightness **0.15 → 0.18**. A real improvement and an honestly modest one; the room is simply dark. Two readings is not a study, and the note is written that way on purpose.

## What is proven and what is not

> **The camera here was a redirected webcam over Remote Desktop**, and the Browser pane blocks camera access by policy — so WebView2 was the *only* place capture could ever be tested. On a physical machine with a local camera both paths should work. Re-confirm a still and the watching mode early; see [[Moving To The New Machine]].

**Proven here:** the intent rules in both directions, the no-camera path (`NotFoundError` → "no camera on this machine"), the refusal path, the *camera on* marker never sticking after a failure, the module only being imported on first use — and, once a webcam was redirected into the VM, **the whole capture path end to end**: host grants the permission, `getUserMedia` succeeds inside WebView2, and a 768×432 frame with genuine detail comes back.

**Still not proven:** no still has yet been sent to Claude. The dev profile's brain has no eyes, and that call costs money — so the last step, the picture actually reaching a model, remains built and unexercised. See [[The Photoreal Decision]] for the same honesty about Stage 8: unverifiable work gets said out loud rather than counted as done.

**Presence detection** — "she notices you arrive" — is not built at all. It needs the camera, so it waits with it.

## There is a camera now *(09/01/2026)*

A spare USB one, plugged into the host. It enumerates as a generic UVC `USB Video Device`, status OK, and carries a microphone of its own. Every note that said the camera path could not be tested on this machine is out of date.

**It has not been opened**, and that is the correct state: `Camera` stays `false` until somebody switches it on in Settings. What has been verified is that the device is present — not that a frame comes back, which is a different claim and needs someone to grant the permission.

One thing that still gates a genuine end-to-end test regardless of hardware: `MaybeLookAsync` returns early unless the brain is `ClaudeBrain`, and there is no API key here. On the `home` profile `look` never fires at all, so the eye button appears and toggles but nothing opens. That is a limitation worth knowing before concluding the camera is broken.

**Its microphone became the default input.** It is now the only capture device Windows lists, and `MicrophoneDevice` is empty — which means "follow the default" — so her ears moved off the Jabra boom without anyone choosing that. `EarsTest -- mic` reports `SIGNAL PRESENT (peak 0.100)`: she can hear, but a camera mic across the desk is not a boom at the mouth. See [[The Ears]] and [[Moving To The New Machine]].

## Turning it on *(v0.21.2)*

**Settings → Let her see you**, with a **Camera** picker beside it. Until v0.21.2 there was no camera control anywhere in the interface: the setting existed, the host handled `setCameraDevice`, `hello` carried the device, and the face received it — but nothing could ever set it, so the only way to enable her camera was to hand-edit `config.json`. The eye button was correctly hidden the whole time, which is why it read as having been removed.

Two things about the picker that are not true of the microphone or output ones:

- **The list comes from the face, not the host.** The camera belongs to whichever renderer the person is sitting in front of — a wall tablet has one, the machine under the desk may not — so `camera.js` enumerates it. Matched by **label**, never by device id: an id is regenerated per origin and per permission grant, so a stored id means nothing tomorrow.
- **It is empty until she has looked once.** A browser withholds device labels from a page that has never been granted the permission, so the menu reads *"Not known yet"* and the hint explains rather than leaving it looking broken.

Enabling the camera is logged at **warn**, disabling at info — a camera coming on in someone's home should leave a mark that is easy to find later.

## Watching — she looks at you

*v0.9.2.* The camera button beside the microphone, wearing the same contract: a person presses it, a red **camera** pill stands beside the state pill the whole time, pressing it again ends it. While it is lit her gaze follows you, which is the difference between talking *at* a screen and being looked at.

Three design decisions:

- **It is renderer-local, completely.** The gaze is a motion centroid computed inside the page at ~8 Hz over a 64×36 grid; no frame, coordinate or flag ever crosses the socket. The host cannot start it, stop it, or know it is happening — which is the strongest privacy statement available, because there is nothing to trust, only nothing to send.
- **It is not a face detector, on purpose.** A detector would be more precise and would bring megabytes of somebody else's model to a job that only has to *feel* attentive. Thirty readable lines instead, and the failure mode is human: when nothing moves, she keeps looking where she last saw you.
- **Only a person can start it.** Code cannot press the button. This is the line between the one-still glance — which she may take when a question needs eyes — and being watched, which is never hers to decide.

The button only appears when `hello` says the host would grant the permission, so it cannot exist in a state where it could only fail.

## The protocol

`look` (host → face) asks for one still. `sight` (face → host) answers with `image` or with `error`.

**Always one or the other.** Silence would leave the host waiting twenty seconds for a frame that is never coming — so a refused permission is an `error`, not nothing. See [[Face Protocol]].
