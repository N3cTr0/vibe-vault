---
project: Octavia
tags: [octavia, code]
source-path: PROTOCOL.md
---

# PROTOCOL.md

```markdown
# Octavia Face Protocol — version 1

The contract between the host (`Octavia.exe`, the being) and a face (a renderer). JSON
messages, each with a `type`. Anything that speaks this protocol is a legal face: the
built-in WebView2 page, a browser on another machine, or — later — an Unreal application
rendering a photoreal character.

The face is a renderer and nothing more. It holds no API key, makes no calls to any model,
and owns no audio device. Everything it knows arrives as a message.

## Transports

| Transport | Used by | Notes |
|---|---|---|
| WebSocket | Any face, including the built-in one | `ws://<host>:<FacePort>/?token=<token>` — or `?key=<remote key>` from another machine |
| HTTP `GET` | Any face that is not the built-in page | The same port serves the page itself. See *Serving the face* below |
| WebView2 postMessage | The built-in page, as a fallback | Automatic if the socket cannot be reached |

The host runs both at once and several faces can be attached simultaneously — useful when
developing a new renderer beside the old one, and the intended arrangement once a tablet is
a face alongside the desktop. Since v0.24.0 a face belongs to a **room**, and most messages
go to a room rather than to everyone; see *Rooms* below.

**The socket binds to `127.0.0.1` unless `RemoteAccess` is on**, in which case it binds
every interface and a remote face must present the remote key. See *Security* below.

## Serving the face

The built-in page reaches `wwwroot` through a WebView2 virtual host mapping, which is a
feature of that control and not a server. Any other renderer needs the files over HTTP, so
**the same port answers plain GETs** that are not WebSocket upgrades:

| Path | Serves |
|---|---|
| `/` | `wwwroot/index.html` |
| `/<file>` | `wwwroot`, including `lib/` |
| `/avatars/<file>` | Her avatars folder — VRM characters, which live in her data folder rather than the install |

Anything outside those two roots is a 404, as is any extension not on the served list.
Paths are checked after resolution, so a symlink is no more use than a `..`.

**Authentication carries in a cookie after the first request.** A page's sub-resources —
`<link href="face.css">`, `import('./watch.js')` — are fetched by the browser, which knows
nothing about a credential, so the first authorised request is answered with an `HttpOnly`,
`SameSite=Strict` cookie and the assets present that. An asset requested without either is
`401`.

**A face served this way should address the socket as the origin it was loaded from**, not
as `127.0.0.1` — that is the host's loopback, not the client's. `?port=` overrides this and
exists for the built-in page, which is *not* served over HTTP and so cannot infer anything.

**`getUserMedia` will not run on a plain `http://<lan-ip>` origin**, because it is not a
secure context. A face on another machine that needs a camera or microphone should own
them natively and answer `sight` itself, rather than expecting a WebView to do it.

## Versioning

The `hello` message carries `protocol`. A face should check it and refuse to run against a
major it does not understand. Within version 1, **new message types and new fields may be
added at any time** — a face must ignore types and fields it does not recognise rather
than failing. Removing or repurposing anything is a version bump.

---

## Rooms (v0.24.0)

**One being, N rooms.** She has one persona, one voice, one avatar, one key and one set of
tools. A *room* is a space she can be talked to in, and it owns what makes a conversation a
conversation: its history, its state, its expression, its floor, and whether a camera may
open in it.

| The being owns | A room owns |
|---|---|
| Persona, voice model, avatar, API key | Its conversation |
| Tools and MCP servers | Its `state` and its `emotion` |
| The host machine's devices and config | Its captions, turns and transcript |
| Her mood *policy* | Her mood *right now* |

**A face names its room in `ready`.** Absent means `host` — the machine she runs on — so the
built-in page and every renderer written before this keep behaving exactly as they did. A
face served over HTTP takes it from its own URL (`?room=phone`), which is how a client puts
two connections in one room by building one query string.

> **Do not derive the room from the credential.** Token-means-loopback-means-host is
> tempting and wrong: two handsets would silently share a room, and a laptop on the LAN
> would be indistinguishable from a phone. A room is a statement of intent; state it.

**A room is not a face.** The Android client opens two connections — a native one that owns
the microphone and a WebView panel that draws her page — and both are the phone's room. If
room and face were the same thing she would be talking to herself in the next tab.

### She attends one room at a time

There is one voice, one Whisper and one turn in flight. A `say` or a held button from
another room while she is mid-turn is **refused out loud**:

```json
{ "type": "notice", "text": "She is talking to someone else." }
```

This is the mechanism `talking` already used for the floor, generalised from *the floor* to
*her attention*. Concurrent rooms are deliberately out of scope: two conversations at once
needs two synthesis pipelines and two transcriptions, and one being holding two
conversations is a worse simulation rather than a better one.

### What a face may drive

Every face→host message is one of three things, and **the host decides by the room the
message came from**, before acting.

| Class | Messages | Rule |
|---|---|---|
| **Host only** | `listen`, `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder`, `saveDiagnostics` | Acted on only from the `host` room. From anywhere else: refused, answered with a `notice`, logged once. |
| **Room** | `say`, `talking`, `hush`, `forget`, `sight`, `setCamera`, `setCameraDevice`, `selfTest`, `faceError` | Acted on for the sending face's room only. |
| **Being** | `setKey`, `setVoice`, `setVoiceEngine`, `setAvatar`, `setRoomHour`, `setStats` | Allowed from any room, and echoed to every room — every face is wearing the result. |

`hello.controls` says which of these a face has (`host` or `room`) so a page can hide what it
cannot use. **That is a hint, not the enforcement.** Both are needed: without the guard a
remote face can send the message by hand anyway, and without the hint a phone shows a
microphone button that silently does nothing.

### Where each message goes

- **To the sending face's room:** `caption`, `turn`, `cleared`, `overheard`, `state`,
  `emotion`, `notice`, `look`, `needKey`, `diagnostics`, `diagnosticsSaved`.
- **To the room she is attending:** `viseme`, and **her voice** — the binary audio frames.
  She speaks in one room, not in every room that once asked for audio.
- **To the `host` room only:** `level` and `music`. Both are measurements of this machine —
  its microphone and its output mix — and a tempo from the speakers under a desk means
  nothing at all to somebody holding a phone in a gym.
- **Per face:** `hello`. It differs by room, by what that face may drive, and by what she is
  doing *there*.

**Her voice does not play where nobody is standing.** When she is attending any room but the
host's, this machine's speakers are silenced for the length of the turn while the visemes and
the streamed PCM continue unchanged. A voice that cannot be streamed at all (`audioAvailable:
false`) therefore goes unheard in a remote room, and the host says so with a `notice` rather
than falling back to speaking aloud in an empty house.

---

## Host to face

| Type | Fields | Meaning |
|---|---|---|
| `hello` | `protocol`, `room`, `controls`, `hasKey`, `model`, `profile`, `ears`, `voice`, `voices[]`, `voiceEngine`, `listening`, `avatar?`, `avatarFile`, `avatars[]`, `roomHour`, `music`, `musicAvailable`, `camera`, `cameraDevice`, `stats`, `microphones[]`, `microphone`, `outputs[]`, `output`, `whisperCompute`, `toolServers[]`, `audioAvailable`, `audioRate`, `audioBits`, `audioChannels`, `micAccepted`, `dev`, `state`, `emotion`, `emotionWeight` | Capabilities and current settings, **for this face**. Sent on connect and whenever they change. `room` is the room this face was put in, echoed back so a typo in `?room=` is visible rather than mysterious; `controls` is `host` or `room` — see *Rooms*. `state`, `emotion`, `camera` and `cameraDevice` are that room's, not the session's. `avatar` is a URL for a VRM character (absent when there is none); `avatars[]` is what the host has to offer and `avatarFile` which is chosen. `voices[]` is `{value, label}` — only the host knows how to tidy a Piper file name into something a menu can show. `microphones[]` and `outputs[]` are `{value, label}` too, where an empty `value` means "follow the Windows default" and is always offered first; the label marks which one that currently is. `toolServers[]` is `{name, ready}` — reported even before she can call them, because "is the integration actually connected" should not need a log to answer. The four `audio*` fields describe her voice as a stream: `audioAvailable` is false on the Windows voice, which cannot be teed, and a face is told so rather than left waiting for frames that were never coming. `audioRate` comes from the live voice model, so it **changes when the voice does** — re-read it on every `hello` rather than caching it once. `micAccepted` says whether the host will take audio *from* a face at all, so a client does not offer a microphone button that could only fail. |
| `state` | `value`: `idle` \| `listening` \| `thinking` \| `speaking` | Her overall state. Drives posture, expression and the status pill. |
| `level` | `value`: 0.0–1.0 | Microphone amplitude, ~20 Hz while listening. This is what makes the face react while you are still talking. |
| `viseme` | `value`: 0.0–1.0, `shape?` | Mouth openness from real phoneme events during synthesis, and which mouth shape to make. Sent at phoneme rate. |
| `emotion` | `value`, `weight`: 0.0–1.0 | The expression she should wear. Sent when it changes, not per sentence. |
| `music` | `beat`, and when `beat` is false also `playing`, `bpm`, `energy` | What the machine is playing, from local analysis of the output mix. See below. |
| `caption` | `who`, `text`, `tentative?` | The line under her face. `tentative` marks a partial transcript. |
| `turn` | `who`: `you` \| `octavia`, `text` | A completed turn, for the transcript log. |
| `overheard` | `text`, `why` | She heard this and decided it was not addressed to her. A face should **show** it, faintly — never swallow it. |
| `look` | — | Take **one** still from the camera and answer with `sight`. Sent only when a question cannot be answered without eyes. Goes to **one** face: one that claims a `camera` in `ready`, in the room that asked. It never leaves that room, and it never reaches a face that said it has no camera. |
| `notice` | `text` | Something the user should read: an error, a model download, a silent microphone. |
| `needKey` | — | She cannot think until an API key is supplied. |
| `cleared` | — | The conversation was forgotten; empty the transcript. |
| `diagnostics` | `running`, `checks[]`, `facts[]`, `log[]` | The result of a self-test. While `running` is true the other fields are absent. |
| `diagnosticsSaved` | `path` | A diagnostics bundle was written there. |

### The `diagnostics` message

`checks[]` is `{ name, ok, detail, fix? }`. **`fix` is present whenever `ok` is false** and
says what to do about it in a sentence — a face should show it, not just the red line.
`facts[]` is `{ name, value }`, ordered for display. `log[]` is the most recent log lines,
oldest first.

`dev` is a hint, not a capability: it says the host is running a development profile and a
face may offer its own debugging tools. A face is free to ignore it, and the host promises
nothing extra when it is true.

`state`, `emotion` and `emotionWeight` are **what she is doing and wearing right now**.
Both are otherwise sent only when they change, so a face attaching to a session already in
progress had no way to learn either — and an expression can sit unchanged for many minutes,
so a renderer that assumed `neutral` was simply wrong until she next felt something. Apply
them on connect, then follow the `state` and `emotion` messages.

## Audio — her voice, on another face

**A binary WebSocket frame is audio and nothing else.** No per-frame header, no type tag.
If a second binary stream is ever wanted that is a version decision, not a field added
quietly. Every JSON message above travels as a text frame; a face can therefore tell the
two apart by frame type alone.

Raw PCM, in the format the four `audio*` fields of `hello` advertise — typically 22050 Hz,
16-bit, mono, but **read it, never assume it**: the rate belongs to the voice model and
changes when the voice does. Base64 inside JSON was rejected: a third more bytes and a
great deal of garbage at ~44 KB/s.

### Asking for it, and why this one is opt-in

```json
{ "type": "subscribe", "skip": ["viseme", "level"], "want": ["audio"] }
```

**No face receives audio until it asks.** That is the opposite of the rule `skip` follows,
and deliberately so: `skip` declines a *rendering hint*, where defaulting to "send it" is
right. Audio is a **physical output**. If it were opt-out, the built-in page and every
browser face on this machine would start playing her voice on top of the speakers she is
already using — not a bandwidth problem but her talking over herself in the same room. A
face that draws her mouth has not thereby claimed the right to make noise.

The host never streams to the built-in page for the same reason.

### Stopping

**A face must throw away buffered audio on any `state` that is not `speaking`.** There is
no separate stop message and there does not need to be one: she has stopped, so anything
still held is a tail she has already finished, and playing it means she carries on talking
on the phone after going quiet in the room. The host drops its own queued frames at the
same moment, so the two halves agree.

### Upstream — a microphone somewhere else

**The same rule applies in reverse: a binary frame from a face is microphone audio and
nothing else.** 16 kHz, 16-bit, mono, little-endian, fixed by contract rather than
negotiated — that is what Silero and Whisper want, and a handset has cycles to spare for the
resample. Only the face currently holding the floor is listened to; frames from anyone else
are dropped rather than mixed, because two rooms transcribed into one sentence is worse than
one of them being ignored.

While a face holds the floor the **local microphone is muted for speech** — but it keeps
running, because her sense of what is *playing in this room* is a different question and
must not follow the phone.

### Back-pressure

A face that cannot keep up **loses the oldest frames**, not the newest, and control
messages are never dropped. Old audio is worthless: a slow face should hear a gap and catch
up rather than fall further behind for the rest of the utterance. A face that stops reading
entirely is eventually dropped.

*No codec. Raw PCM is what a LAN behind Wireguard can afford and it costs nothing to
implement. Opus is the obvious later optimisation — roughly a tenth the bytes — and it
belongs between the tee and the queue.*

### The `music` message

Two shapes, and a face has to expect both:

- **`{ type: "music", beat: true }`** and nothing else — a beat, right now. Sent on its own
  the instant one is detected, and never coalesced with anything, because a beat that
  arrives late is worse than one that never arrived.
- **`{ type: "music", beat: false, playing, bpm, energy }`** — the state. `playing` is
  whether this is music rather than speech or a notification; `bpm` is 0 until a tempo has
  been found; `energy` is 0–1. Sent at roughly twelve a second and only when something
  moved, like `level`.

A face should drive its movement from the `beat` messages rather than from a clock of its
own running at `bpm`. The host is the one listening; a face that predicted its own beats
would drift out of time with the music within a few bars.

**No audio crosses the protocol, and none is kept.** The host analyses the output mix
locally and sends these three numbers. There is no message that carries sound.

### `look` and `sight` — the camera

**The face owns the camera, and the host owns the decision.** The face is where the
person is: a wall tablet has a camera pointed at the room, the machine under a desk may
not. But a renderer must never decide to open one.

A conforming face:

- opens the device **only** on `look`, never on a timer and never speculatively
- takes **one** frame and stops the track in the same breath — there is no streaming mode
- **shows that the camera is live**, unmistakably, for as long as it is
- always answers, with `image` or with `error`. A refused permission is an `error`, not
  silence

The host sends `look` only when three things are true: the camera is enabled in config,
the words genuinely need eyes, and the brain has any. None of those consults a model —
the decision to open a camera in someone's home is one a person must be able to audit by
reading it.

**No frame is stored.** It rides with the one question that asked for it and is never
written into the conversation history.

`camera` in `hello` says whether the host would grant the permission at all, so a face can
hide controls that could only fail.

**Watching — her following you with her eyes — is not in this protocol on purpose.** It
is a renderer-local mode: a person presses the face's camera button, a motion centroid is
computed inside the page, the gaze moves, and nothing — no frame, no coordinate, no flag —
crosses the socket. The host cannot start it, stop it, or know it is happening. A face
that offers it must make it person-started and visibly marked for as long as it runs.

### The face vocabulary

`emotion.value` is one of **`neutral`, `happy`, `angry`, `sad`, `relaxed`, `surprised`**.
`viseme.shape` is one of **`aa`, `ih`, `ou`, `ee`, `oh`**, or absent for a closed mouth.

These are not our names — they are the **VRM 1.0 expression presets**, chosen so that a
real character needs no translation layer between the protocol and its blendshapes. A
face with no blendshapes maps them to whatever it has; a face with a VRM passes them
straight through.

`shape` is an addition, so a face written before it must keep working from `value` alone.

### Rates and ordering

`level` and `viseme` are high-frequency and lossy by nature — a face that misses one
should simply use the next. Everything else is meaningful and ordered. `level` is only
sent while listening, `viseme` only while speaking; a face should decay `level` on its own
if the stream stops, rather than freezing the last value.

---

## Face to host

| Type | Fields | Meaning |
|---|---|---|
| `ready` | `faceBuilt`: bool, `room?`: string, `senses?`: string[] | The face has loaded. `faceBuilt` is false if the renderer failed to construct — the host logs this. Triggers `hello`. `room` names the space this face is in; absent means `host`, so nothing written before rooms existed changes behaviour. `senses` is what this face can actually do — `["mic", "camera"]` — and is how `look` finds a face with a camera instead of guessing at whoever last spoke. **An absent `senses` is not an empty one**: it means a face that predates the field, and such a face is still asked. A face that claims nothing is a renderer, which is a legal face and always has been. |
| `subscribe` | `skip`: string[], `want`: string[] | Message types this face does not want, and streams it does. See *Audio* below for `want`. Answered by the socket server itself and never relayed to the host, because it describes one connection rather than the session. **Opt-out, not opt-in**: a face that never sends it keeps receiving everything, so no existing renderer changes behaviour and a new message type reaches old clients rather than being silently withheld. A phone would send `{"skip":["viseme","level"]}` — sixty visemes a second is a battery, not a feature. `want` is the counterpart and is **opt-in**; the two are independent, and being sent audio is not a reason to start receiving visemes again. |
| `say` | `text` | The user typed something. |
| `talking` | `value`: bool | **Push-to-talk from a face.** True takes the floor and begins a binary audio stream; false releases it, and is the end-of-utterance marker — so the voice detector never has to guess where the sentence stopped. Deliberately **not** `listen`, which toggles *her own* microphone and has to keep working independently. One face holds the floor at a time and a second press is refused with a `notice`. A face that disconnects mid-stream counts as a release, and there is a timeout, because a phone in a pocket must not own her ears for ever. Pressing while she is speaking **hushes her**, which is what talking over someone means. |
| `listen` | — | Toggle listening. |
| `hush` | — | Stop speaking and abandon the current reply. |
| `forget` | — | Clear the conversation. |
| `setKey` | `value` | Store an API key. Written in, never read back out. |
| `setVoice` | `value` | Select a voice by name, from the list in `hello`. |
| `setVoiceEngine` | `value`: `windows` \| `neural` | Choose the speech engine. The neural one may have to download before it can speak; she keeps talking with the old one until it is ready. |
| `setAvatar` | `value` | Choose a character by file name; empty means the plaster bust. The host refuses a name that is not in its avatars folder. |
| `setRoomHour` | `value`: 0–23, or −1 | Pin the room's lighting to an hour, or follow the clock. |
| `setMusic` | `value`: bool | Whether she listens to the machine's output at all. False closes the loopback device rather than merely ignoring it. |
| `setMicrophone` | `value` | Capture device by name, from `microphones[]`. Empty follows the Windows default. |
| `setOutput` | `value` | Render device by name, from `outputs[]`. Empty follows the Windows default. This is what her music sense listens to, so it is not merely a playback preference. |
| `setCamera` | `value`: bool | Whether she may open a camera **in the sending face's room**. "May she look at all" is a question about a place, not about her: a phone in a gym and a desk should be able to answer it differently. Off by default and the only sense that is; the host echoes it back as `camera` in `hello`, which is what un-hides the eye button. Only the host room's answer is written to the config file, because that file belongs to this machine. Enabling is logged at **warn** — a camera coming on in someone's home should leave a mark that is easy to find later. |
| `setCameraDevice` | `value` | Which camera a `look` should open, by **label** rather than id — a device id is regenerated per origin and per permission grant, so it cannot be stored in a config file and still mean anything tomorrow. Empty lets the face choose. |
| `setWhisperCompute` | `value`: `auto` \| a named backend | Which compute Whisper should use. `auto` is the default and the right answer on any machine nobody has measured. |
| `setStats` | `value`: bool | Whether the face should show its own performance figures. A hint to the renderer, stored by the host so it survives a reload. |
| `sight` | `image?`, `error?` | The answer to `look`: one base64 JPEG, or why there isn't one. **Always send one or the other** — silence leaves the host waiting for a frame that is not coming. |
| `faceError` | `text` | The renderer threw. The host writes it to `octavia.log`, since a face has no console the host can see. |
| `selfTest` | — | Run the checks and answer with `diagnostics`. Free — it never calls a paid model. |
| `saveDiagnostics` | — | Ask the host for a bundle. **The host owns the file dialog**; a face never chooses the path. |
| `openDataFolder` | — | Show her data folder in Explorer. |

---

## Security

A local WebSocket is reachable by anything running on the machine, so:

- The listener binds **`127.0.0.1` by default** — never `0.0.0.0`. Nothing off-machine can
  reach it without a deliberate tunnel.

### Remote faces (v0.14.0, off by default)

`RemoteAccess: true` binds every interface as well, so a phone or a wall tablet can
attach. It changes the authentication rules rather than relaxing them:

| Where from | Accepts |
|---|---|
| Loopback | the per-run `token`, **or** the remote key |
| Anywhere else | the remote **key only**, and only when `RemoteAccess` is on |

The per-run token is deliberately *not* accepted remotely: it is written to the log and
carried in a URL, and it is scoped to a process on this machine. The remote key lives in
`<data>\remote.key`, survives restarts — a phone cannot re-pair every time she starts —
and is presented as `?key=...`. Regenerating it revokes every device at once, which is
the whole revocation story: there is no per-device list.

> **The key path did not work at all until v0.23.1.** A length guard no generated key could
> satisfy meant every read of the key replaced it, so a remote face was always compared
> against a secret a microsecond old. It went unseen for nine versions because the phone
> reached her over `adb reverse` — loopback, and therefore the token. The first WiFi
> connection hit it immediately. Anything written against the remote key before 0.23.1 was
> written against a path nothing had ever walked.

**The key is one shared secret in front of a microphone and, later, a house.** That is
enough behind Tailscale or Wireguard, where "every interface" means the tailnet and the
LAN. It is *not* enough behind a forwarded port, and the host logs a warning saying so
every time remote access starts.
- Every connection must present a **token** generated fresh at each start:
  `ws://127.0.0.1:<port>/?token=<token>`. A connection without it is refused at the
  handshake with `401` and never becomes a WebSocket.
- The token is written to `octavia.log` at startup so an external face can be pointed at
  her; it changes every run.
- The built-in page receives it through its own URL, which is served from the host's
  virtual origin and never leaves the machine.

The key itself never crosses the protocol outward: `setKey` carries it *in*, and `hello`
reports only a boolean.

`saveDiagnostics` asks for a bundle but cannot say **where**: the host raises its own file
dialog and a person picks the path. A face that could name the destination would be a face
that could write a file containing the log anywhere on the machine.

**This is a speed bump, not a security boundary.** It stops a stray page or careless
script on the same machine from driving her; it is not a defence against a local attacker
who can already read the log.

## Reconnection

The host does not push state on a timer. A face that reconnects gets a fresh `hello`,
which carries **what persists** — her state and her expression — so a renderer is correct
from its first frame rather than guessing.

What is *not* replayed is what has already happened: `caption` and the transcript. A
reconnecting face should start with an empty placard and let the next turn fill it.

## What a renderer must implement

The minimum to be a legal face, and the checklist an alternative renderer is built
against. `tools\attach-face.ps1 -Conformance` drives a running host through a turn and
reports which of these actually arrived, so the contract is verified rather than believed.

| Must handle | Because |
|---|---|
| `hello` | Everything else is meaningless without the vocabulary version and the current state |
| `state` | Her posture and the status readout |
| `viseme` | The mouth. `shape` may be absent — fall back to openness alone |
| `emotion` | The face she wears. The current one arrives in `hello` |
| `caption`, `turn` | What is being said, and the transcript |
| `notice` | Errors and progress a person has to read |
| `overheard` | Otherwise she appears to ignore people at random |

| May ignore | Consequence |
|---|---|
| `level` | She stops reacting while you speak, but still answers |
| `music` | She never dances. Everything else works |
| `diagnostics`, `diagnosticsSaved` | No health panel; the host still logs and still writes bundles |
| `look` | She has no eyes. Answer it with a `sight` carrying an `error`, never with silence |
| `needKey`, `cleared` | Degraded, not broken |

**Must send:** `ready` on load. Everything else in the face-to-host table is optional —
a renderer with no controls is a perfectly legal face.

**Rates:** `viseme` arrives at phoneme rate and `level` at about 20 Hz while listening;
both are lossy by nature, so a renderer that misses one should use the next rather than
buffering. `music { beat: true }` is the exception — it is rare, and it is worthless late.

## Planned additions (still version 1)

Nothing outstanding. `music` landed in v0.8.0 and is documented above.
```
