---
project: Octavia
tags: [octavia, spec, stage-14]
---

# Stage 14, item 3 — Her voice, out of another face

> **A specification, not a change.** Written 08/31/2026 from the [[Octavia Android]] side, which is the consumer that needs it, for whoever is working in her repo. Nothing in her repo was touched to produce it. Companion to [[Stage 14 - Face Identity]], which landed in v0.21.0. The reasoning lives in [[Roadmap]] under *Stage 14 — More than one body*.

## Why this one next

**Today her voice cannot leave this machine.** `NeuralVoice` writes to a `WaveOut` and `SapiVoice` calls `SetOutputToDefaultAudioDevice()`. A tablet in another room sees her mouth move in silence.

It is the right item to take before the microphone (item 2) for three reasons: it is the smaller change, it is independently useful — a tablet becomes worth *looking at* before it is worth talking to — and it does **not** depend on turn ownership (item 5), because a face that wants her voice should get it regardless of who spoke.

It does not depend on item 1 either, strictly: the per-connection state it needs sits beside `Skip`, which already existed. Item 1 being done simply makes the logging clearer.

## Do this first, or audio will not work at all

`WebSocketFaceServer.Broadcast` fans out like this:

```csharp
foreach (var (id, face) in _faces)
{
    …
    _ = SendAsync(id, face.Socket, bytes);   // un-awaited, one per face
}
```

**`WebSocket.SendAsync` does not permit two concurrent sends on the same socket.** A second call while one is outstanding throws `InvalidOperationException: There is already one outstanding 'SendAsync' call`. Fire-and-forget means nothing serialises them.

This is already a latent race — visemes go out at phoneme rate and can overlap a `caption` or a `level`. It is *rare* because each send completes in microseconds on loopback. **At roughly forty audio frames a second to a handset over Wireguard it stops being rare and becomes the normal case.**

### And the failure is silent, which is the worse half

`SendAsync` catches **every** exception and removes the face, without logging — deliberately, and for a good reason at the time:

```csharp
catch (Exception)
{
    // A face that went away mid-send is not an error worth logging every frame;
    // viseme messages would flood the log.
    _faces.TryRemove(id, out _);
}
```

So a concurrent-send `InvalidOperationException` does not surface as an error. It **silently drops a live, healthy face.** Its socket stays open and its receive loop keeps running, so the client believes it is still connected — it simply never receives another message. From the sofa that reads as *"she went quiet"*, with nothing in the log to say why, and it would clear on reconnect, which is the signature of a bug that takes weeks to pin down.

Worth separating the two cases while you are in here: a face that genuinely went away is routine and should stay quiet; **a send that failed for any other reason should be logged once.** The distinction is cheap and it is the difference between this being diagnosable and not.

So item 3 begins with a **per-face send queue**:

- One `Channel<Outbound>` per `Face`, and one writer loop draining it. Every send becomes an enqueue; ordering is then guaranteed and concurrency is impossible by construction.
- **Bound it, and drop rather than grow.** A face that cannot keep up should hear a gap and catch up, not fall further behind for the rest of the utterance. Dropping the *oldest* audio is right — old audio is worthless — while control messages (`state`, `caption`, `turn`) should never be dropped. Two priorities, or a small queue for audio and an unbounded one for the rest.
- A face whose queue is persistently full is a face that has effectively gone; the existing removal path can take it.

This is worth doing even if the rest of item 3 is deferred.

## The design

### 1. Audio is opt-**in**, and that is a deliberate exception

`subscribe` is opt-*out* on purpose: a new message type should reach old faces rather than be silently withheld. **Audio must not follow that rule.**

If audio were opt-out, the built-in page and every browser face on this machine would immediately start playing her voice **on top of the speakers she is already using**. That is not a bandwidth problem, it is her talking over herself in the same room.

The distinction worth writing into `PROTOCOL.md`: `skip` declines a *rendering* hint; audio is a *physical output*, and a face that draws her mouth has not thereby claimed the right to make noise.

Extend the existing message rather than adding a second mechanism:

```json
{ "type": "subscribe", "skip": ["viseme", "level"], "want": ["audio"] }
```

`want` is the counterpart to `skip`, for streams that are expensive or physically exclusive. Absent means absent — no face receives audio until it asks.

### 2. Binary frames, and binary means audio

Base64 inside JSON is a third more bytes and a great deal of garbage at ~44 KB/s. Send PCM as **binary WebSocket frames**.

The server currently sends `WebSocketMessageType.Text` only (one call site, line ~302), so this is new plumbing — small, but real.

**Rule: a binary frame is audio and nothing else.** No per-frame header, no type tag. If a second binary stream is ever wanted, that is a protocol version decision, not a header field added quietly.

### 3. The format is announced, never assumed

`NeuralVoice` builds `new WaveFormat(_sampleRate, 16, 1)` and **`_sampleRate` is read from the voice's own config** — 22050 for Amy, but it varies per voice and changes when the voice changes.

So `hello` gains:

| Field | Meaning |
|---|---|
| `audioAvailable` | Whether this engine can stream at all |
| `audioRate` | Samples per second, from the live voice |
| `audioBits` | 16 |
| `audioChannels` | 1 |

`hello` is already re-sent whenever settings change, so a voice change carries the new rate with it. A face must re-read the format on every `hello` rather than caching it once.

### 4. Where to tee — the part that is already done

`NeuralVoice.OnAudioPlayed(ReadOnlySpan<byte> pcm)` is handed **exactly what goes to the sound card, at the moment it goes there**. That is not a coincidence: it is how the visemes stay in step with what is heard, via `MouthTap`.

Teeing there gets audio that is in sync with the visemes a face is already receiving, for free, at the one point in the codebase where that is guaranteed. Anywhere earlier and it drifts.

**One trap:** `OnAudioPlayed` receives a `Span`, which cannot be captured by an event handler or an async send. The tee must **copy** — take from `ArrayPool<byte>.Shared`, copy, enqueue, return to the pool once written. Do not hand the span onward.

Suggested shape on `IVoice`, beside the existing `Viseme`:

```csharp
/// The audio as it reaches the sound card. Raw PCM in the format `hello` advertises.
/// Raised only while something is subscribed, so a host with no remote face pays nothing.
event Action<ReadOnlyMemory<byte>>? Audio;
```

### 5. SAPI is out of scope, and says so

`SapiVoice` calls `SetOutputToDefaultAudioDevice()`; getting its PCM means `SetOutputToAudioStream` and a rework. **Do not attempt it here.** `audioAvailable: false` on that engine means a client is *told* the voice cannot be streamed, rather than being left waiting in silence — which is the whole difference between a limitation and a bug.

### 6. Stopping

When she is hushed mid-utterance, a face holding buffered audio must throw it away, or she carries on talking on the phone after stopping in the room.

No new message: **a face must flush its audio buffer on any `state` that is not `speaking`.** Add that as a rule in `PROTOCOL.md`. Confirm on the way that `Hush()` does land in `Finished` → a non-speaking `state`; if it does not, that is the actual bug to fix and it is a small one.

## Explicitly not in scope

- **No microphone.** Audio *upstream* is item 2 and is the larger half.
- **No turn ownership.** Audio goes to whoever asked for it, not to a designated face. Item 5 does not gate this and should not be pulled in.
- **No codec.** Raw PCM, because the target is a LAN behind Wireguard and it costs nothing to implement. Opus is the obvious later optimisation — roughly a tenth the bytes — and the place it would go is between the tee and the queue. Say so in a comment; do not build it.
- **No echo cancellation.** That belongs to item 6 and only matters once the phone also has a microphone.

## Acceptance

1. A face that has **not** sent `want: ["audio"]` receives **no binary frames at all**. Check this first: it is the one that stops her talking over herself.
2. A face that asks receives PCM, and playing it back gives intelligible speech at the rate `hello` advertised.
3. `hello` reports `audioAvailable: false` on the Windows voice, and true on the neural one.
4. Changing voice mid-session re-sends `hello` with the new `audioRate`.
5. **Concurrency:** with a face subscribed to audio *and* another attached without `skip`, a full utterance completes with no `InvalidOperationException` in the log. This is the send-queue fix being proven.
6. `hush` mid-sentence stops the remote audio promptly, with no tail.
7. A face that stops reading is dropped rather than growing the host's memory.

`tools\EarsTest` is the right home for 1, 5 and 7 — two real socket faces, one subscribed and one not, exactly as `FaceProtocolChecks` already does for `look`.

## What the phone will do with it

This is the item that makes a tablet worth putting on a wall. [[Octavia Android]] will:

- send `want: ["audio"]` only when it is the device meant to speak, which is a user setting rather than a default;
- feed frames to an `AudioTrack` in streaming mode at the advertised rate;
- flush on any non-`speaking` state, per the rule above;
- keep skipping `viseme` and `level` — it has no mouth to move yet, and **being sent audio is not a reason to start receiving those**. (v0.21.0's `SendTo` already honours `skip` for exactly this reason.)

## Links

- [[Stage 14 - Face Identity]] — item 1, landed in v0.21.0
- [[Roadmap]] — *Stage 14*, for the other items and the ordering
- [[The Voice]] — how she speaks today
- [[Octavia Android]] — the client that needs this
