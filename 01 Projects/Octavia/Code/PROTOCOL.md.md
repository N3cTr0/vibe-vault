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

The host runs both at once and broadcasts to every connected face, so several can be
attached simultaneously — useful when developing a new renderer beside the old one, and
the intended arrangement once a tablet is a face alongside the desktop.

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

## Host to face

| Type | Fields | Meaning |
|---|---|---|
| `hello` | `protocol`, `hasKey`, `model`, `profile`, `ears`, `voice`, `voices[]`, `voiceEngine`, `listening`, `avatar?`, `avatarFile`, `avatars[]`, `roomHour`, `music`, `musicAvailable`, `camera`, `cameraDevice`, `stats`, `microphones[]`, `microphone`, `outputs[]`, `output`, `whisperCompute`, `toolServers[]`, `dev`, `state`, `emotion`, `emotionWeight` | Capabilities and current settings. Sent on connect and whenever they change. `avatar` is a URL for a VRM character (absent when there is none); `avatars[]` is what the host has to offer and `avatarFile` which is chosen. `voices[]` is `{value, label}` — only the host knows how to tidy a Piper file name into something a menu can show. `microphones[]` and `outputs[]` are `{value, label}` too, where an empty `value` means "follow the Windows default" and is always offered first; the label marks which one that currently is. `toolServers[]` is `{name, ready}` — reported even before she can call them, because "is the integration actually connected" should not need a log to answer. |
| `state` | `value`: `idle` \| `listening` \| `thinking` \| `speaking` | Her overall state. Drives posture, expression and the status pill. |
| `level` | `value`: 0.0–1.0 | Microphone amplitude, ~20 Hz while listening. This is what makes the face react while you are still talking. |
| `viseme` | `value`: 0.0–1.0, `shape?` | Mouth openness from real phoneme events during synthesis, and which mouth shape to make. Sent at phoneme rate. |
| `emotion` | `value`, `weight`: 0.0–1.0 | The expression she should wear. Sent when it changes, not per sentence. |
| `music` | `beat`, and when `beat` is false also `playing`, `bpm`, `energy` | What the machine is playing, from local analysis of the output mix. See below. |
| `caption` | `who`, `text`, `tentative?` | The line under her face. `tentative` marks a partial transcript. |
| `turn` | `who`: `you` \| `octavia`, `text` | A completed turn, for the transcript log. |
| `overheard` | `text`, `why` | She heard this and decided it was not addressed to her. A face should **show** it, faintly — never swallow it. |
| `look` | — | Take **one** still from the camera and answer with `sight`. Sent only when a question cannot be answered without eyes. |
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
| `ready` | `faceBuilt`: bool | The face has loaded. `faceBuilt` is false if the renderer failed to construct — the host logs this. Triggers `hello`. |
| `subscribe` | `skip`: string[] | Message types this face does not want. Answered by the socket server itself and never relayed to the host, because it describes one connection rather than the session. **Opt-out, not opt-in**: a face that never sends it keeps receiving everything, so no existing renderer changes behaviour and a new message type reaches old clients rather than being silently withheld. A phone would send `{"skip":["viseme","level"]}` — sixty visemes a second is a battery, not a feature. |
| `say` | `text` | The user typed something. |
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
