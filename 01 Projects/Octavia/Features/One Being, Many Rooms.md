---
project: Octavia
tags: [octavia, feature]
---

# One Being, Many Rooms

*Stage 14 item 9, v0.24.0.* She stops being one screen with several windows onto it.

Built from [[Stage 14 - Two Rooms]] — a specification written on the [[Octavia Android]] side by the client that needed it, for whoever was working in her repo. It replaced item 5, absorbed item 7, and struck item 4 as already done.

## The ask

> On the phone I should not be able to toggle the host mic/keyboard. Say one day I am at the gym and accidentally click it and no one is at home — it is a pointless thing. **One brain, same avatar, same personality, but different spaces.**

## Two faults, and they are worth keeping apart

One was five lines and security-shaped. The other was the architecture.

### A phone could drive this machine

`listen` toggles **the host's microphone**. The handset renders the same page as the desktop, so pressing the mic button in a gym opened a microphone in an empty house. `PROTOCOL.md` was honest that this is deliberately not a room-level control — and nothing enforced it: **no `set*` case looked at where the message came from at all**. The same was true of `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder` and `saveDiagnostics`.

Hiding the button was never the fix. A face that can send `listen` by hand can still open the microphone.

### There was one conversation

`RespondTo` did not take a face. `caption`, `turn`, `state`, `emotion` and `cleared` all went out with no target, which means everyone. `_brain` held the one `Conversation` there was, and `Forget()` cleared it. So typing at her on the phone put your words on the desktop's screen and played her answer in a room you were not standing in.

## The model

**One being. N rooms.** The being owns what makes her *her*; a room owns what makes a conversation a conversation.

| The being owns | A room owns |
|---|---|
| Persona, voice model, avatar, API key | Its `Conversation` |
| Tools and MCP servers | Its `state` and its `emotion` |
| This machine's devices and config | Its captions, turns and transcript |
| Her mood *policy* | Her mood *right now* |

**Emotion is per room on purpose.** It drives the avatar, and a global mood would put an expression on the phone's face that was caused by a conversation happening in a different building — which is precisely the thing being complained about. Same personality, different mood, the way a person is.

**A room is not a face.** The Android app opens *two* connections: a native client that owns the microphone, and a WebView panel that draws her page. Both are the phone's room. If room were face, the app's own panel would become a third room and she would be talking to herself in the next tab.

## The authority table

Every face→host message is one of three things, and the check is on the sending face's room, in `OnFaceMessage`, before the switch acts.

| Class | Messages | Rule |
|---|---|---|
| **Host only** | `listen`, `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder`, `saveDiagnostics` | Host room only. Elsewhere: refused, answered with a `notice`, logged once per room per kind. |
| **Room** | `say`, `talking`, `hush`, `forget`, `sight`, `setCamera`, `setCameraDevice`, `selfTest`, `faceError` | The sending face's room. |
| **Being** | `setKey`, `setVoice`, `setVoiceEngine`, `setAvatar`, `setRoomHour`, `setStats` | Any room, echoed to every room — every face is wearing the result. |

`hello` gained `controls` (`host` or `room`) and the page hides its host-only rows accordingly. **That is a hint, not the enforcement**, and both halves are needed: without the guard a remote face drives the hardware anyway, and without the hint a phone shows a microphone button that silently does nothing, which is its own kind of broken.

**Refusing beats ignoring.** A face that quietly does nothing looks broken, and somebody will spend an evening on it.

> **`setCamera` moved from being-wide to per room.** "May she open a camera at all" is a question about a *place*, not about her. The gym phone and the desk answer it separately, and only the host room's answer is written to the config file — because that file belongs to this machine. Enabling is still logged at **warn**, and the line names the room. See [[Eyes]].

## She attends one room at a time

One voice, one Whisper, one `_responding` flag, one cancellation source. A `say` or a held button from another room while she is mid-turn is **refused out loud**:

```json
{ "type": "notice", "text": "She is talking to someone else." }
```

This is not a new mechanism. It is exactly what `TakeFloor` already did when a second face pressed the button — generalised from *the floor* to *her attention*. See [[The Ears]].

**Concurrent rooms are explicitly out of scope**, and not only because two Whispers and two synthesis pipelines on eight cores is a much larger change. It is *untrue to the thing being modelled*: one being cannot hold two conversations at once, and pretending otherwise is a worse simulation, not a better one. If it is ever wanted, it should be a separate decision made against a measured machine.

## Her voice, which was the one actively wrong

`SendAudio` reached every face that had opted in, so she spoke aloud in rooms nobody was standing in — a question asked on a handset was answered out loud at an empty desk, and in two rooms at once if the desk had asked for audio.

It takes a room now. And when she is attending any room but this one, **this machine's speakers are silenced for the length of the turn**.

Silenced at the sound card and nowhere earlier. The visemes and the streamed PCM are read from the same buffer at the same instant — that is the whole reason the tap sits where it does, see [[The Voice]] — so a phone still receives her voice in step with her mouth, and only the room she is not in goes quiet. `MouthTap` asks whether she should be audible *after* the tap and zeroes the buffer; returning nothing would stall playback and take her mouth and her state with it.

The Windows voice cannot be streamed at all, so a remote room is *told* — a `notice` saying she can be read and not heard — rather than her falling back to talking out loud in an empty house.

## Where each message goes

- **The sending face's room:** `caption`, `turn`, `cleared`, `overheard`, `state`, `emotion`, `notice`, `look`, `needKey`, `diagnostics`.
- **The room she is attending:** `viseme`, and her voice.
- **The host room only:** `level` and `music`. Both measure *this machine* — its microphone and its output mix — and a tempo from the speakers under a desk means nothing to somebody holding a phone in a gym. This is the room-microphone trap from [[Stage 14 - A Microphone Somewhere Else]] wearing different clothes.
- **Per face:** `hello`, which it never was before. It had already drawn blood once — the avatar URL had to be patched in the renderer because one `hello` could not say different things to the built-in page and a phone.

## Two seams that moved

**`Conversation` came out of `IBrain`.** There are N conversations and one of her, and a `ClaudeBrain` per room would duplicate the HTTP client and the key in order to hold two lists of strings apart. `RespondAsync` takes the history instead — which `Conversation.cs` was already shaped for, its own header saying it is "kept provider-neutral so both brains share one shape". Forgetting is `history.Clear()`, in the room that asked. See [[The Brain]].

**`_lastSpokenThrough` is gone**, as its own comment asked it to be. A face declares `senses` in `ready` and `look` goes to one that claims a camera, in the room that asked. On Android this is not a nicety: the native client owns the camera and the WebView panel cannot open one at all, because `getUserMedia` needs a secure context and the panel is served over plain HTTP. Without `senses` the host had a coin-flip chance of asking the half of the phone that physically cannot answer.

> **An absent `senses` is not an empty one.** It means a face written before the field existed, and such a face is a candidate of last resort rather than a refusal. That distinction is the whole of why `attach-face.ps1`, `EarsTest` and the built-in page needed no changes.

## Nothing existing changed

A face that names no room is in the host room. The desktop page, `attach-face.ps1` and the checks all pass untouched — the same instinct as `subscribe` being opt-*out*.

> **The room is not derived from the credential.** Token-means-loopback-means-host is tempting and wrong: two handsets would silently share one room, and a laptop on the LAN would be indistinguishable from a phone. A room is a statement of intent and is stated — in `ready`, from `?room=` on the URL a face was loaded with, which is how the Android app puts its two connections in one room by building one query string.

## What is proven, and what is not

`EarsTest -- rooms` drives a real `OctaviaSession` through a recording transport and a forty-line stub of a local model, and asserts all ten of the spec's acceptance criteria in process — no handset, no API key. Forty-seven assertions.

Each of the four mechanisms was broken on purpose first, to watch the right checks go red: broadcasting again turned six red, removing the authority table eleven, sharing one `Conversation` one, and always speaking aloud one. The conversations in the check run between two **non-host** rooms deliberately, so the suite proves the routing without the machine talking out loud on every run.

**And then a real one turned up.** Three seconds after the first build with rooms in it started, a handset at `10.1.1.181` authorised with the remote key and both of its connections — the native client, which asked for audio and skipped visemes, and its WebView panel — landed in room `phone` together. The Android side had been sending `ready.room` against the spec before the host understood the field, so until that build it was simply ignored. **Acceptance criterion 8 was confirmed on real hardware without anybody arranging it**, which is a better result than the check that asserts the same thing. See [[Screenshots]] v0.24.0.

**And criterion 7 closed on hardware too**, the same day, from the Android side. A probe face joined room `phone` declaring `senses: []`, leaving the native client as the only camera there — so `look: asking face a85b541d in room 'phone'` → `sight: 1280x960, brightness 0.57` → `got a frame, 97 KB`, and the WebView panel was never asked. It had been held open here on an inherited belief that there was no API key on this machine, which was **false**; the default profile is simply a local brain. See [[Eyes]] and [[Changelog]] 0.24.1.

**Nothing is now owed on either side.** The in-process check still asserts the *choice of face* — the part a handset cannot easily prove, since it needs a second face in the room that deliberately has no camera — and it runs on a local brain, which is why it asserts the rule rather than driving a turn.

## The thing this cost, and how it was paid *(v0.25.0)*

Hiding the two host-only controls was right and it had a price nobody costed at the time: a handset ended up **less capable than the machine it was standing in for**, on a device that has a microphone and a camera and whose native client already owns both.

Neither could be given back on the wire. The floor is a `FaceId`, so the WebView panel cannot press while the native connection streams — and making the floor room-scoped instead would let any face in a room feed her ears, which is a worse rule than the one there is. Watching is renderer-local by design and should stay there.

So the page borrows from whatever it is embedded in. See [[Lending A Renderer The Device's Senses]] — and note that a **borrowed camera is deliberately not claimed to the host**, because `senses` routes `look` and the embedder lends gaze rather than stills.

> **The lesson is about the shape of the fix, not the fix.** Item 9's guard was correct and produced a real capability gap two versions later. A rule that says *no* to a whole class of face will strand something legitimate inside that class, and the answer is a seam that lets the legitimate case identify itself — not a hole in the rule.

### And it stranded one more thing, invisibly *(v0.25.1)*

`listen` was doing **two jobs**, and only one of them was a host-room concern:

| Job | Whose |
|---|---|
| Open this machine's microphone | The **host room's** device. Locking it was right. |
| Start the speech recogniser | **Being-wide** — the same Whisper for every room. |

Item 9 locked the first and took the second with it. `TakeFloor` required a running recogniser, only `listen` started one, and `listen` was now refused from the only rooms that needed it — so **a room face could never start her ears**. It was invisible for two versions because no remote face had a microphone button to press until [[Lending A Renderer The Device's Senses]] gave it one back, and then it was reported from the handset immediately: `micAccepted: false, ears: not started`.

Holding the button now opens her ears. A held button is an explicit request to be transcribed, which is precisely the meaning `listen` was also carrying.

> **Three things would have gone wrong quietly while fixing it**, and all three are the same shape — a call that does more than its name. `UseSource` *starts* what it is given, so handing the ears back the local microphone on release would have opened the desk's microphone because a phone let go of a button. `Start` subscribed without detaching, so a face taking the floor on ears the desk already had would have processed every frame twice. And a mute set during her reply was only ever lifted when the *desk* was listening, so the second press would have been open, correctly sourced, and silent.

**`micAccepted` also meant the wrong thing** — "already open" rather than "will accept" — so a handset was told its button could only fail, hid it correctly, and the fault read as a client bug.

## Links

- [[Stage 14 - Two Rooms]] — the specification this was built from
- [[Conventions & Security Model]] — the authority table as a security posture
- [[Face Protocol]] — `room`, `senses`, `controls` on the wire
- [[The Attention Gate]] — one per room, with no shared statics
