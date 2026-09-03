---
project: Octavia
tags: [octavia, octavia-android, spec, stage-14]
---

# Stage 14 item 6 — Always-on listening in a room

> Written from the Android side on 09/02/2026, against her **v0.26.1** and client **v0.10.0**,
> in the same way [[Stage 14 - Two Rooms]] and [[Stage 14 - Lending A Renderer The Device's Senses]]
> were. It is the last difference between the Windows client and the phone.

> [!done] Built the same day, in her v0.28.0 and client v0.12.0.
> Confirmed on the 11T Pro: her name spoken into a room she is listening to reaches her with
> nothing pressed, and **74 seconds of her own voice into an open microphone produced no
> utterance at all** — the gate held back 3226 frames while she spoke.
>
> Three things the build taught that this spec did not predict, kept below under
> *What it actually cost*.

## What item 6 actually says

> *"`Mute()`/`Unmute()` around her speech is what stops her transcribing herself, and it works
> because everything is in one process on one clock. A tablet with an open mic and a speaker
> playing her voice, both across a network with latency in each direction, will hear her and
> transcribe it."*

So the mic button is **held** on a room face and a **toggle** on the host, and her placard on a
handset says *"Press the microphone, or say her name"* — where saying her name does nothing at
all. Item 6 is what makes that sentence true.

**Stage 15 item 3 raised the stakes.** Once the server holds no device and the Windows client
streams its microphone exactly as the phone does, the desktop inherits this problem. Item 6
stops being the last polish on a remote room and becomes a prerequisite for the desktop
keeping always-on listening at all.

## What was measured first, on the actual handsets

`dumpsys media.audio_flinger`, 09/02/2026:

| Device | Acoustic Echo Canceler | Noise Suppression |
|---|---|---|
| Xiaomi 11T Pro (`vili`) | **Qualcomm Fluence**, `libqcomvoiceprocessing.so` | Qualcomm Fluence |
| Galaxy J7 Pro (SM-J730F) | NXP Software Ltd (software) | NXP Software Ltd |

Two things follow, and they are the whole design:

1. **The good AEC is real but conditional.** `libqcomvoiceprocessing` attaches only for
   `VOICE_COMMUNICATION` input. The client currently opens `VOICE_RECOGNITION`, which is the
   better source for a recogniser and the one source that will *never* get Fluence.
2. **It cannot be depended on.** The J7's is a generic software canceller on an Exynos, and
   `AcousticEchoCanceler.isAvailable()` returning true says nothing about how well it works.
   A design that requires AEC is a design that works on one handset.

## The design

**Gating is the workhorse; echo cancellation is the bonus that buys barge-in.**

### The client half — [[Octavia Android]]

**1. Half-duplex gating, driven by the client's own playback.**

The host knows when it *sends* her audio. It does not know when the handset's speaker actually
emits it, nor when it stops — buffering and jitter sit in between, which is exactly why the
in-process `Mute()`/`Unmute()` does not transfer.

**The client knows both, exactly.** It owns the `AudioTrack`. So the suppression belongs here,
not in the protocol: while her voice is playing, and for a short tail after the buffer drains,
microphone frames are not sent. No clock sync, no protocol change, and it works on every device
including one with no AEC at all.

This is also the standing constraint working as written: reflex-speed things stay local.

**2. Platform echo cancellation where the device has it.**

For always-on only, open the microphone as `VOICE_COMMUNICATION` and attach
`AcousticEchoCanceler` and `NoiseSuppressor` when available. This is what makes **barge-in**
possible — interrupting her mid-sentence, which pure gating forbids by construction.

**Push-to-talk keeps `VOICE_RECOGNITION`.** It is the better source for Whisper, and a held
button has no echo problem worth paying AGC and telephony noise-shaping for.

Where AEC is absent or poor, gating still holds and barge-in is simply not offered. The
device decides which it gets; neither is a failure.

**3. The button becomes a toggle**, matching the host, and the placard becomes true.

### The host half — this repo

**1. `listen` from a room means "this face streams continuously".**

Today `listen` is `HostOnly` and means *open the host machine's microphone*. Stage 15 item 3
already decided it becomes a room concern. A room face asking to listen should open **its own**
stream, never a device on the server.

**2. A held floor is the wrong mechanism, and this is the hard part.**

`_floor` is a single `FaceId`, `FloorLimit` is 60 seconds, and rooms are serialised. A face
that simply held the floor forever would starve the desktop and time out anyway. Always-on
needs a room to be **open** — streaming and being transcribed — while still yielding between
utterances, so the other room can speak.

This is the part to design carefully rather than bolt on: it touches the most-debugged code in
the project.

**3. The attention gate must apply to always-on and must NOT apply to push-to-talk.**

> **A bug found while specifying this, and it is the reason item 6 cannot be purely a client
> change.**
>
> `OctaviaSession.cs` says, in `Consider`:
>
> > *"In practice only the host room ever reaches here: push-to-talk bypasses the gate
> > entirely, since a held button has already answered the question it asks."*
>
> **It does not.** `EarsRoom` resolves to the floor holder's room, `OnHeard` is the single sink
> for everything the recogniser produces whatever source it was given, and `Consider` judges
> with `RoomNamed(EarsRoom).Gate`. So a held button on the phone **is** judged — and the gate
> drops anything under twelve characters as *"too short to be addressed to anyone"*.
>
> Press the button, say *"yes"*, and she ignores you. The gate is on by default (`Gate =
> "local"`).
>
> **Why it was never caught:** `RoomChecks` constructs its session with `Gate = "off"`. The one
> harness that drives a face taking the floor has the gate disabled, so the interaction it
> would have exercised is the interaction it cannot see.
>
> Item 6 has to separate the two anyway — always-on without a gate is unusable, push-to-talk
> with one drops deliberate speech — so this is fixed as part of it rather than before it.

## Acceptance

1. A room face can turn always-on listening **on and off** and is not refused.
2. With it on, speaking her name in that room reaches her; the placard is true.
3. **She does not transcribe herself.** Her voice playing on the handset produces no utterance,
   with the speaker at a normal level and the handset on a desk.
4. **Push-to-talk still bypasses the gate.** A held button and the single word *"yes"* reaches
   her, where the same word overheard does not.
5. **The other room is not starved.** With the phone listening continuously, the desktop can
   still take a turn.
6. Turning it off returns the client to exactly today's push-to-talk behaviour, including the
   audio source.
7. On a device with no usable AEC, 3 still holds — gating alone is sufficient — and barge-in is
   absent rather than broken.

## What it actually cost

Three things the design above did not see coming, all found on the handset.

**1. `listen` had to travel through the embedder, not the page.** The host binds *"the face
that is streaming"* to whoever asked. Sent from `bridge.js` it would have named the panel — a
face with no microphone — and every frame the shell then sent would have been dropped as
coming from somebody else. Exactly the trap `talking` already solves, and the same answer.

**2. The gate must never swallow a press.** The first build gated on
`listeningOn && voice.audible`, which is right until somebody holds the button *while she is
speaking* — which is precisely when they most mean it, and why `startTalking` hushes her. A
held button and the word "yes" reached her as *nothing voiced at all*. The condition needs
`&& !pressing`, and the doc comment above the code had said so all along.

**3. A listening room has to look like one.** Only `#watch` had a pressed style, so a
microphone left listening looked identical to one switched off — and turning it off looked
exactly like the tap not working, which is how it was first reported. The host was also never
setting the room's state, so the pill read `idle` while she was actively listening. Both are
the interface disagreeing with the microphone, which is the one disagreement none of this is
allowed to have.

**And a fourth, which is not item 6's fault but is item 6's problem.** With a room left
listening, the most likely way an utterance now dies is somebody speaking over the end of a
long answer: `OnHeard` opened with a bare `if (_responding) return;` and discarded it in
silence. Rooms are serialised and that is correct; being silent about it is not. It says so
now.

## What this deliberately does not do

- **No wake-word engine.** Her name is matched by the attention gate on a transcript, which is
  what the host room already does. A local wake-word model is a different piece of work.
- **No always-on when the app is in the background.** Stage 7 keeps her connected, not
  listening; a microphone open behind a screen you are not looking at needs a person's explicit
  consent and a marker, exactly as the camera does.
- **No concurrent rooms.** Rooms stay serialised.

## Links

- [[Stage 14 - Two Rooms]] · [[Stage 14 - Lending A Renderer The Device's Senses]] · [[Stage 15 - Server And Clients]]
- [[Octavia]] · [[Octavia Android]]
