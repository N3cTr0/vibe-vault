---
project: Octavia
tags: [octavia, spec, stage-14]
---

# Stage 14, item 1 — Faces need identity

> **A specification, not a change.** Written 08/31/2026 from the [[Octavia Android]] side, which is the consumer that needs this, for whoever is working in her repo. Nothing in her repo was touched to produce it. The reasoning lives in [[Roadmap]] under *Stage 14 — More than one body*; this note is the implementation detail.

## Why this is first

`IFaceTransport` is broadcast-only in **both** directions. `Send(object message)` goes to everyone, and `event Action<JsonElement>? MessageReceived` says **nothing about who sent it**.

That was a deliberate simplification — "the being stops knowing about the renderer" — and it is exactly right while every face only *watches*. It stops being right the moment a face has a microphone, a camera, or a turn.

Every other Stage 14 item depends on this one, so it goes first and alone.

**It has already drawn blood twice**, which is the argument for doing it properly rather than working around it again:

1. **The avatar URL had to be rewritten in the renderer** (v0.20.0). `hello` is serialised *once* and broadcast to every attached face, so the host cannot tell the built-in page `https://octavia.avatar/x.vrm` and a phone `/avatars/x.vrm`. The face now patches its own URL — correct given the constraint, and a workaround for this missing seam.
2. **`look` opens every camera at once.** See *The bug this fixes*, below.

## The bug this fixes, concretely

`OctaviaSession` asks for eyes with a broadcast and waits on a single promise:

- `_face.Send(new { type = "look" })` — line ~851, goes to **every** attached face
- `private TaskCompletionSource<string?>? _looking` — line 825, **one** slot
- `case "sight":` — line ~348, `_looking?.TrySetResult(…)`

With a tablet and the desktop both attached, asking her something that needs eyes **opens both cameras**, lights the privacy marker in an empty room, and the first answer to arrive wins — arbitrarily. That quietly breaks the second promise `camera.js` makes about itself in its own header: *"It is never opened unasked."*

## The change

### 1. Two small types

```csharp
namespace Octavia.Face;

/// Which face. Opaque to the session: it may compare, store and route with one,
/// and can learn nothing from it about how that face connected.
internal readonly record struct FaceId(Guid Value)
{
    public static FaceId New() => new(Guid.NewGuid());

    /// Short form, for the log. A full GUID per line is noise.
    public override string ToString() => Value.ToString("N")[..8];
}

/// One inbound message and the face it came from.
internal readonly record struct FaceMessage(FaceId From, JsonElement Body);
```

### 2. `IFaceTransport`

```csharp
event Action<FaceMessage>? MessageReceived;          // was Action<JsonElement>
void Send(object message, FaceId? to = null);        // was Send(object)
```

**`to` is optional and `null` keeps today's meaning: everyone.** This is the whole reason the change is small — 24 of the 25 send sites are unchanged, and a new message type still reaches every face by default rather than being silently withheld. Same instinct as `subscribe` being opt-*out*.

### 3. `WebSocketFaceServer`

Most of this already exists. `_faces` is `ConcurrentDictionary<Guid, Face>` and `ServeAsync` already mints `var id = Guid.NewGuid()` — the identity is there, it simply never leaves the class.

- Key `_faces` by `FaceId`.
- Raise `MessageReceived` with `new FaceMessage(id, body)`.
- Add `SendTo(FaceId id, string json, string? type)` — same `Skip` check as `Broadcast`, one recipient.
- `Broadcast` unchanged.

### 4. `WebViewFaceTransport`

Mint one `FaceId` at construction and expose it. The built-in page is a face like any other and needs an id to be addressable; it just never changes.

### 5. `FaceHub`

```csharp
public void Send(object message, FaceId? to = null)
{
    var json = JsonSerializer.Serialize(message, Json);
    var type = message.GetType().GetProperty("type")?.GetValue(message) as string;

    if (to is null)                       { _page?.SendJson(json); _sockets?.Broadcast(json, type); return; }
    if (to == _page?.Id)                  { _page.SendJson(json); return; }
    _sockets?.SendTo(to.Value, json, type);
}
```

Still serialised once. `Relay` wraps the inbound element with the sender's id.

### 6. `OctaviaSession`

- `OnFaceMessage(FaceMessage m)` — read the body from `m.Body`; nothing else in the `switch` changes.
- `_looking` becomes `(FaceId Face, TaskCompletionSource<string?> Waiting)?`.
- `case "sight":` **ignores a frame from a face that was not asked**, and logs it rather than swallowing it. A face answering a `look` it never received is a bug worth seeing.
- The `look` send targets one face.

### Which face gets the `look`?

Proper turn ownership is **item 5 and is not in scope here.** The honest interim, which fixes the real bug without pretending to be more:

> Track the `FaceId` of the last inbound message that came from a *person* — `say`, `listen`, `hush`. Send `look` there. Fall back to the built-in page when there has been none, which is the case when the utterance came from the PC's own microphone.

One field. It is not turn ownership and the comment should say so.

## Explicitly not in scope

Items 2–7 of Stage 14, and they should not be smuggled in:

- **No audio**, up or down. That is items 2 and 3 and it is the larger work.
- **No voice routing.** `NeuralVoice` still writes to its `WaveOut`.
- **No turn ownership.** The `look` target above is a stopgap and should be replaced by item 5, not grown into it.
- **No protocol change.** Nothing here is visible on the wire, so `PROTOCOL.md` does not move. That is a good sign: this is a seam being repaired, not an interface being widened.

## Acceptance

Testable, and the first two are the point of the whole item:

1. **Two faces attached, ask something needing eyes → exactly one camera opens.** The other shows no privacy marker.
2. **A `sight` from a face that was not asked is ignored and logged.**
3. `hello`, `state`, `caption`, `turn`, `overheard`, `notice` still reach **every** attached face.
4. `subscribe` / `skip` still applies per connection.
5. The built-in page still works with the socket unbound, over the `postMessage` fallback.
6. `Attached` in `FaceStatus` and the diagnostics count are unchanged.

A second face is cheap to get for testing: `http://127.0.0.1:<FacePort>/?token=<token>` in any browser since v0.20.0, and the token is in her log. Two browser tabs are two faces.

## What the phone does with it

**Nothing, immediately** — and that is the point. This unblocks rather than delivers:

- The tablet stops opening its camera for questions asked at the desk.
- It is the prerequisite for her voice reaching the phone (item 3) and the phone's microphone reaching her (item 2), because both have to be addressed to *one* face.
- `hello` could eventually be tailored per face, which would retire the avatar-URL rewrite in `bridge.js`.

## Links

- [[Roadmap]] — *Stage 14 — More than one body*, for the reasoning and the other six items
- [[Octavia Android]] — the client that needs this
- [[Face Protocol]] — unchanged by this work
- [[Architecture]] — the seam being repaired
