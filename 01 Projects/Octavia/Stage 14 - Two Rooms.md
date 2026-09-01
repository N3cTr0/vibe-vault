---
project: Octavia
tags: [octavia, spec, stage-14]
---

# Stage 14, item 9 — Two rooms

> **Built, 09/01/2026, v0.24.0.** Added by the repo side, below the original text rather than inside it — everything under this line is as it was written. All ten acceptance criteria are asserted in `EarsTest -- rooms` against a real session, with criterion 7 the one exception: `look` needs the Claude brain and there is no key on this machine, so the *choice of face* is checked directly and the round trip is still owed. Both traps were real and both are commented in the code. See [[One Being, Many Rooms]] and [[Changelog]] 0.24.0.

> **Criterion 7 closed, 09/01/2026, from the [[Octavia Android]] side.** `look` → `sight` walked end to end with the frame off the handset: `look: asking face a85b541d in room 'phone'` → `sight: 1280x960` → `got a frame, 97 KB`, matching `CameraStill: one frame, 97 KB` in the phone's own log. The asking face declared **no senses**, so the native client was the only camera in the room, and the WebView panel was never asked — `senses` doing the job it was added for. **The API key was not missing.** Item 1 recorded "no key on this machine" and item 9 carried that forward; `data\apikey.dat` decrypts under this user to a 108-character `sk-ant-…`, and `--profile cloud` starts her on `brain: claude-sonnet-5`. It needed a restart, not a secret. Nothing is now owed on either side.

> **A specification, not a change.** Written 09/01/2026 from the [[Octavia Android]] side, which is the consumer that needs this, for whoever is working in her repo. Nothing in her repo was touched to produce it. It supersedes item 5 (*turn ownership*) and absorbs item 7 (*the attention gate now has two rooms*) — see *What this replaces*.

## The ask, in the owner's words

> On the phone/tablet, I should not be able to toggle the host mic/keyboard. Say one day I am at the gym and accidentally click it and no one is at home — it is a pointless thing. The goal is that she is **independent of the host interface**. She should not be doing what the host is doing. **1 brain, same avatar, same personality, but different spaces** — so what goes on in one room shouldn't reflect on the other face.

## What is actually wrong today

Two separate faults, and they are worth keeping apart because one is a five-line guard and the other is the architecture.

### 1. A remote face can drive the host's hardware

`bridge.js:552` is the whole bug:

```js
el('talk').addEventListener('click', () => send({ type: 'listen' }));
```

`listen` toggles **the host machine's microphone** (`OctaviaSession.cs:341` → `ToggleListening()`). The phone renders the same page as the desktop, so the mic button on a handset at the gym opens the microphone in an empty house. `PROTOCOL.md` is honest about this — *"Deliberately not `listen`, which toggles her own microphone"* — but nothing enforces it, and **no `set*` case in `OctaviaSession` looks at `inbound.From` at all.** The same applies to `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder` and `saveDiagnostics`: all of them act on the machine she runs on, and all of them are reachable from a phone.

This half is a security-shaped problem, not a UI one. Hiding the button is not the fix; the host must refuse the message.

### 2. There is one conversation, and every face is a window onto it

- `RespondTo(string userText)` (`OctaviaSession.cs:1098`) does not take a face. `case "say"` throws `inbound.From` away.
- `caption` and `turn` (`:1117`, `:1118`), `state` (`:1213`), `emotion` (`:1200`), `cleared` (`:351`) all go out with no target, which `FaceHub.Send` (`FaceHub.cs:56`) documents as meaning *everyone*.
- `_brain` is a single instance holding a single `Conversation`, and `Forget()` clears the only one there is.
- `SendAudio` (`FaceHub.cs:38`) broadcasts her voice to **every** face that opted in, so she speaks in both rooms at once.

So today, typing at her on the phone puts your words on the desktop's screen, and her answer plays in a room you are not in. That is the "different spaces" half, and it is a real piece of work.

## The model

**One being. N rooms.** The being owns what makes her *her*; a room owns what makes a conversation a conversation.

| The being owns | A room owns |
|---|---|
| Persona, voice model, avatar, API key | Its `Conversation` |
| Tools and MCP servers | Its `state` and its `emotion` |
| The host machine's devices and config | Its captions, turns and transcript |
| The Whisper model, the Piper voice | Its floor, its microphone, its camera |
| Her mood *policy* | Her mood *right now* |

**A room is not a face.** This matters immediately: the Android app already opens **two** connections — the native client that owns the microphone, and the WebView panel that draws her page. Both are the phone's room and must stay in step. If room were face, the app's own panel would become a third room and she would be talking to herself in the next tab.

> **Emotion is per room on purpose.** It drives the avatar. Global mood would put an expression on the phone's face that was caused by a conversation happening in a different building, which is precisely the thing being complained about. Same personality, different mood — the same way a person is.

## The decision that keeps this small

**Rooms are serialised. She attends one room at a time.**

She has one Piper voice, one Whisper, one `_responding` flag and one `_turn` cancellation source. Two rooms conversing *simultaneously* means two synthesis pipelines, two transcriptions on an eight-core box, and two brains in flight. That is a much larger change, and it is also **untrue to the thing being modelled** — one being cannot hold two conversations at once, and pretending otherwise is a worse simulation, not a better one.

So a `say` or a held button from room B while she is mid-turn in room A is **refused out loud**:

```json
{ "type": "notice", "text": "She is talking to someone else." }
```

This is not a new idea — it is exactly what `TakeFloor` already does when a second face presses the button (`OctaviaSession.cs:966`), and that mechanism is proven. Generalise it from *the floor* to *her attention*.

Concurrent rooms are **explicitly out of scope** and should be a separate decision made against a measured machine, not assumed into this one.

## The change, host side

### 1. Rooms exist, and a face declares one

```csharp
namespace Octavia.Face;

/// Which space a conversation is happening in. Faces in the same room see the same
/// conversation; faces in different rooms share nothing but her.
internal readonly record struct RoomId(string Value)
{
    /// The machine she runs on. Faces that name no room land here, so the built-in
    /// page and every existing renderer keep today's behaviour exactly.
    public static readonly RoomId Host = new("host");

    public override string ToString() => Value;
}
```

**A face names its room in `ready`**, which already reaches the session and already triggers `hello`:

```json
{ "type": "ready", "faceBuilt": true, "room": "phone" }
```

Absent means `host`. That default is the same instinct as `subscribe` being opt-out: **no existing renderer changes behaviour**, and the desktop page needs no edit to keep working.

For a face served over HTTP, the room comes from the URL it was loaded with — `?room=phone` — so the Android app can put its WebView panel in the same room as its native connection by building one query string. `bridge.js` reads `location.search`, which it already does for `?port=`.

> **Do not derive the room from the credential.** It is tempting — token means loopback means host — but two handsets would then silently share one room, and a laptop on the LAN would be indistinguishable from a phone. Rooms are a statement of intent and should be stated.

### 2. Faces declare what they can do

The stopgap `_lastSpokenThrough` (`OctaviaSession.cs:1039`) exists because `look` had to pick a face and nothing said which face had eyes. Its own comment says it *"must not grow into"* turn ownership. Retire it:

```json
{ "type": "ready", "faceBuilt": true, "room": "phone", "senses": ["mic", "camera"] }
```

`look` then goes to a face **in the asking room that claims a camera**, rather than to whoever last spoke. This matters concretely on Android: the native client has a camera and the WebView panel does not — `getUserMedia` cannot run on a plain `http://<lan-ip>` origin, which `PROTOCOL.md` already states. Without `senses`, the host has a 50% chance of asking the half of the phone that physically cannot answer.

A face that claims nothing is a renderer, which is a legal face and always has been.

### 3. An authority table, enforced in the session

Every face→host message falls into one of three classes. **The check is on `inbound.From`'s room, in `OctaviaSession.OnFaceMessage`, before the switch acts** — not in the renderer.

| Class | Messages | Rule |
|---|---|---|
| **Host only** | `listen`, `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder`, `saveDiagnostics` | Acted on only from a face in `RoomId.Host`. From anywhere else: refused, answered with a `notice`, logged once at info. |
| **Room** | `say`, `talking`, `hush`, `forget`, `sight`, `setCamera`, `setCameraDevice`, `selfTest`, `faceError` | Acted on for the sending face's room only. |
| **Being** | `setKey`, `setVoice`, `setVoiceEngine`, `setAvatar`, `setRoomHour`, `setStats` | Allowed from any room — these are *her*, not the room. Changing them is echoed to every room, because every face is now wearing it. |

> **`setCamera` moves from being-wide to per-room.** "May she open a camera at all" is a question about a place, not about her. The gym phone and the desk should be able to answer it differently, and the host still logs enabling at **warn** as it does today.

Refusing rather than ignoring matters: a face that silently does nothing looks broken, and someone will spend an evening on it.

### 4. Routing

- `caption`, `turn`, `cleared`, `overheard`, `state`, `emotion`, `notice`, `look`, `needKey` → the room they belong to. `FaceHub.Send` already takes a target; it gains a room-shaped overload, or `SendToRoom(object, RoomId)` beside it.
- `hello` → **per face**, which it is not today. `Announce()` (`OctaviaSession.cs:497`) builds one anonymous object and broadcasts it. This has already drawn blood once: the avatar URL had to be patched in the renderer because one `hello` could not say different things to the built-in page and a phone (her ROADMAP records this under Stage 14 item 1). It needs to differ per face anyway now — see `controls` below.
- `level`, `viseme`, `music` → the room she is currently speaking or listening in. `music` is the **host's** room only; it comes from the host machine's output mix and means nothing on a phone in a gym.
- **Her voice** — `SendAudio` must take a room. This is the one that is actively wrong today rather than merely coarse: she currently speaks aloud in every room that asked for audio, including empty ones.

### 5. `hello` gains two fields

| Field | Meaning |
|---|---|
| `room` | The room this face was put in. Echoed back so a face can show it and so a typo is visible rather than mysterious. |
| `controls` | `"host"` or `"room"`. What this face may drive. A page hides its host-only controls when this is `"room"`. |

`controls` is a **hint for the UI, not the enforcement**. The enforcement is item 3. Both are needed: without the guard a remote face can still send the message by hand, and without the hint the phone shows a mic button that does nothing, which is its own kind of broken.

### 6. The attention gate, per room *(absorbs item 7)*

`AttentionGate` answers *"was that addressed to me?"*. With two rooms it needs to answer it per room, and its state — recent context, whether she was just spoken to — cannot be shared or the desk's conversation will make the phone's gate think it is mid-exchange.

Note that push-to-talk **bypasses the gate entirely** and always has: a held button has already answered the question. So this item only bites when a room gets always-on listening, which on Android it does not have and is not getting in this piece of work. **Scope it, do not build it** — one gate instance per room, constructed with the room, and no shared statics.

## What this replaces

- **Item 5 (turn ownership)** — subsumed. "Which face is she talking to" becomes "which room is she attending", which is a better question: it survives a room having two faces in it, which is the actual arrangement on Android.
- **Item 7 (the attention gate has two rooms)** — folded in above.
- **Item 4 (camera arbitration)** — already landed in v0.21.0; the ROADMAP entry is stale and can be struck through. `look` names a face today. This spec changes *how the face is chosen*, from "last spoken through" to "has a camera, in the asking room".
- **`_lastSpokenThrough`** — deleted, as its own comment asks.

## Traps

### 1. The built-in page must not become a special case

It is tempting to write `if (face == BuiltInFace)` for the host-only check. Do not: a browser on the same machine, `attach-face.ps1`, and `EarsTest` are all legitimately in the host room without being the built-in page, and a future second desktop face would be too. **The check is on the room, and the built-in page is simply always in `RoomId.Host`.**

### 2. `Forget()` is on `IBrain`, and there is one brain

`IBrain.Forget()` clears the conversation the brain owns. With rooms there are N conversations and one brain. The clean move is to **lift `Conversation` out of the brain** and pass it into `RespondAsync`, which `Conversation.cs` is already shaped for — its header says it is *"kept provider-neutral so both brains share one shape and neither owns the trimming policy"*. Constructing a whole `ClaudeBrain` per room instead would duplicate the HTTP client and the key for no reason.

### 3. The room-music analyser belongs to the host room

This is the same trap that [[Stage 14 - A Microphone Somewhere Else]] hit and it will present differently this time. `MusicWatcher` listens to the host machine's **output mix**. When she starts speaking only into the phone's room, the host's loopback still hears her voice through its own speakers if audio is also playing there — and a room's `music` state must not be sent to a room that cannot hear it. Send `music` to `RoomId.Host` only, and revisit if a phone ever grows an ambient sense of its own.

### 4. One `_responding` flag is load-bearing and must stay that way

The temptation while touching `RespondTo` is to make it re-entrant "since there are rooms now". That is the concurrency change this spec explicitly defers. Keep the single flag, add the room it belongs to, and refuse the other room out loud.

## Acceptance

Testable from the Android side, which is where the second room actually is.

1. The phone's mic button **cannot** open the host's microphone. Verified by sending `listen` from the phone by hand and seeing a refusal in `octavia.log`, not a state change.
2. Typing at her on the phone puts **nothing** on the desktop's placard or transcript.
3. Her answer to a phone question plays **only** through the phone. The desktop stays silent — check the log for the audio route, not just the room.
4. Her state pill on the desktop stays `idle` for the whole of a phone conversation.
5. `forget` from the phone clears the phone's transcript and leaves the desktop's intact.
6. A `say` from the phone while the desktop is mid-answer is refused with a `notice`, and the desktop's turn completes undisturbed.
7. `look` reaches the phone's **native** client, never its WebView panel. Verified from `octavia.log` naming the face id, and from the native client being the thing that raises the camera permission.
8. Both of the phone's connections — native and panel — receive the same room's messages, and the panel renders the phone's conversation rather than the desktop's.
9. A face that sends no `room` still behaves exactly as it does today. The desktop page, `attach-face.ps1` and `EarsTest` need no changes to pass.
10. Enabling the camera from the phone is logged at `warn`, and does not enable it on the desktop.

## Explicitly not in scope

- **Concurrent rooms.** She attends one at a time. See *The decision that keeps this small*.
- **Always-on listening in a remote room**, and therefore real echo cancellation. Push-to-talk stands, as item 6 already decided.
- **Cross-room awareness.** She does not know, and is not told, that the other room exists. Whether she *should* — "your wife asked me the same thing this morning" — is a genuinely interesting question and a completely different one.
- **Persisting a room's conversation across a restart.** Rooms are in memory, like the conversation is today.
- **More than two rooms in practice.** The design carries N; nothing needs to be tested beyond two.

## What the phone will do

Tracked on the Android side, listed here so the contract is visible from both ends:

- Send `room` and `senses` on `ready`, from both connections, and pass `?room=` to the WebView panel so its page joins the same room.
- Own the camera **natively** — CameraX, one still, stopped in the same breath, with a visible and unmistakable live indicator for as long as it is open, answering `sight` with an `image` or an `error` and never with silence.
- Hide its host-only controls when `hello` says `controls: "room"`.

## Links

- [[Stage 14 - Face Identity]] — item 1, the seam everything here routes through
- [[Stage 14 - A Microphone Somewhere Else]] — item 2, the floor mechanism this generalises
- [[Stage 14 - Her Voice On Another Face]] — item 3, the audio path that now needs a room
- [[Octavia Android]] — the client this is for
