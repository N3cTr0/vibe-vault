---
project: Octavia
tags: [octavia, code]
source-path: versions.md
---

# versions.md

```markdown
# Octavia — Version History

Pre-release `0.x` scheme. **PATCH by default** (`0.x.y`) for fixes, tweaks and copy
changes; MINOR (`0.x.0`) only for a new subsystem or a notable behaviour change.
Roadmap stages are substantial by definition, so each completed stage takes a MINOR.

Headers use ISO `YYYY-MM-DD` — an internal doc convention, separate from displayed
dates, which are `MM/DD/YYYY`.

---

## 0.28.3 — 2026-09-02

**She can see the network.** The first real tool server: `tools\unifi-mcp.ps1`, five
read-only tools over the UDM SE's own local Integration API, plugged into the seam Stage 12
built in v0.22.0 and never had a real server for.

**No Home Assistant is involved, and the roadmap said there would have to be.** That advice
is still right about the *house* — Google Home has no API a Windows program can use — and
wrong about the network: the gateway answers an official local API itself. UniFi Network
Integration v1 (application `10.6.101`) for sites, devices, clients and statistics, and
**UniFi Protect answers the same key** (application `7.2.105`) for cameras.

`list_devices`, `list_clients`, `get_status`, `find_client`, `list_cameras`. **`list_clients`
is presence** — eighteen named clients, `Kitchen - Plug - Microwave` and the like — which
answers "is anyone home" with no smart-home integration existing yet.

Output is written for a model rather than a parser: `UDM SE: online, up 13d 1h` and
`CPU 5%, memory 63%`, not the raw JSON, which spends most of its tokens on identifiers
nothing downstream reads. Failures come back as text for the same reason the seam already
required — a model told *"the gateway could not be reached"* can say so, where an exception
only ends the turn.

**A 401 is not proof that an endpoint exists.** The first probe read `401` on the integration
path and took it as confirmation the API was enabled; a nonsense path under `/proxy/network/`
returns `401` too, because the proxy answers before it routes. Only a real key settled it.

**Both Protect cameras are offline** — `Front Door` and `Back Garden`, both `DISCONNECTED`,
and a snapshot returns `503` rather than an image, so that state is real rather than a stale
flag. `list_cameras` reports what is *reachable* rather than what exists, because "she has
cameras" and "she can see anything" are separate claims.

Network and Protect are one server rather than two, departing from the roadmap's "each
integration independently broken-able": they are two applications on one appliance behind one
address and one key, so when the UDM is unreachable both are and there is no independence to
preserve.

Not built: the camera snapshot. Protect returns a JPEG and MCP can carry an image, but the
brain-side loop does not exist *and* neither camera is online — two blockers, and writing it
against neither would be guessing.

`EarsTest -- unifi` drives the real gateway and **skips** when no key is configured, so it
stays green on a machine with no house. Eight assertions; the one worth having is *"every
tool is judged a read"*, because `RiskOf` checks its dangerous words first and a description
that gained a `restart`, a `reset` or an `order` would quietly turn a status query into
something she stops to ask permission for. Broken on purpose to watch it go red. 298 pass.

**She still cannot call any of them.** The brain-side tool loop is the open half of Stage 12,
and it now has something real and entirely harmless to be written against — which was the
whole argument for doing UniFi before the house.

---


## 0.28.2 — 2026-09-02

**Her shortcut did not start her.** Stage 15 split her into two processes and left the
desktop shortcut written the day before pointing at only one of them. Double-clicking
`Octavia.exe` with no server running gave a blank window for thirty seconds and then a
message blaming a script syntax error — the rare cause, named first, because the message
predated the split and had been correct when this process *was* the server.

The client now starts her: if her address is on this machine and nothing answers, it runs
`Octavia.Server.exe`, waits for the port, and only then loads a page.

**The reconnection in `bridge.js` could never have covered this.** It recovers a socket that
dropped; it cannot recover a page that was never served, because the retry lives in a file
that has to be downloaded from the thing that is missing. A server that goes away and a
server that was never there are different faults with different answers.

Her console is started minimised rather than hidden, deliberately: on Windows the close
button is what raises SIGTERM, and a server with no console at all can only be killed. The
taskbar entry is also the honest thing to show — there really are two processes now.

The failure message can finally name one cause instead of guessing between two, because the
client knows whether anything answered before it ever loaded a page. Missing server, dead
server, wrong machine and genuine page fault now read differently, and the first two no
longer cost thirty seconds to say.

**What stopping her taught, in three measurements and two lies.** `CloseMainWindow` returns
true and the process is gone in six seconds with her shutdown handler never run — nothing
logged, her sound card released by the OS instead of by her. Ctrl+C is delivered and ignored;
a server started by a window does not answer to one. Ctrl+Break works, arrives as SIGQUIT and
logs cleanly every time — and cannot be survived by whoever raises it, since
`SetConsoleCtrlHandler(NULL, TRUE)` ignores Ctrl+C only and a real handler returning true did
not save the caller either. So the client does not stop her at all. She is stopped by her own
console, and by the SCM once Stage 15 item 4 is built.

That item stays open and stays wanted. It was argued here that a client which can start her
made the service unnecessary, on the grounds that the server and the client always share a
box; **the owner corrected that** — *"it may not always be the case"* — and the correction is
worth more than the claim. A deployment is the one thing in this project guaranteed to
change, and item 3 exists precisely to stop the code believing in one. This is the on-demand
half of item 4, not a replacement for it.

Also struck a fourth stale note that item 6 overtook, inside item 3, where it would have
misled whoever picked that item up: the desktop does not inherit the phone's echo problem
when it starts streaming like a phone. It inherits the phone's answer.

---


## 0.28.1 — 2026-09-02

**She was too small on a phone**, and the framing rule says why. It fits her *width*: a
narrower viewport shows less across at a given distance, so the camera retreats in proportion.
Right down to about square, and wrong past it. A handset in portrait is about **0.63**, which
asked the camera to stand back **2.25×** — more than twice the desktop's retreat — and she
arrived as a small figure adrift in a tall empty room.

She is a tall, narrow subject. Below square the limit stops being her width and becomes the
height of the screen, and no amount of backing away improves that. The aspect is clamped at
both ends now, so anything narrower than 0.9 is framed as if it were 0.9 — which crops nothing,
because there is nothing beside her.

**The desktop is untouched**, and measurably so: its stage sits around 1.85 and still clamps to
1.3, for the same `fit` of 1.09 it always had. Only portrait moved.

## 0.28.0 — 2026-09-02

**Stage 14 item 6: a room can be left listening.** The last thing the Windows client could do
that a handset could not, and the last item of Stage 14. Specced first, in the vault as
*Stage 14 — Always-On Listening In A Room*.

Her placard has said *"Press the microphone, or say her name"* on every face for months. On a
phone the second half was a lie until now.

### `listen` means two different things

From the desk it is *"open the microphone on the machine she runs on"*, and that is hers to
protect — it is why the authority table exists at all. From a room it is *"transcribe what I
am already sending you"*: a claim about the sender's own device, with nothing here to refuse.
So it is answered before the host-only table rather than inside it, because it stopped being a
question about her hardware. Stage 15 item 3 concludes it should mean only the second thing
everywhere, once the server holds no device.

### Being listened to is not holding the floor

`_floor` is a single holder with a sixty-second limit. A face that simply held it would starve
the desk and then time out, so a room being *open* is a separate, quieter claim: `_openFace`
and `_openRoom`, no timer, and audio accepted from either claim.

**A press still works on top of it and hands back afterwards.** Pressing the button inside a
room that is listening must not switch off the thing somebody deliberately left on — so
`ReleaseFloor` returns to the open stream instead of stopping the ears, and `TakeFloor` reuses
the source that is already streaming rather than building a second one and going deaf.

One room at a time. She has one recogniser, one source and one voice, which is the same
serialisation turns already live under, and the second asker is told out loud.

### The echo is answered on the client, and it has to be

`Mute()`/`Unmute()` works at the desk because everything is on one clock. The host knows when
it *sent* her voice; it does not know when a handset's speaker emitted it, nor when it
stopped. **The client knows both exactly**, because it owns the track — so the suppression
lives there and nothing about it crosses the socket. See the Android client's v0.12.0.

Measured: 74 seconds of her own voice into an open microphone produced no utterance at all.

### Two things that were true and invisible

**A listening room did not look like one.** Only the watch button had a pressed state, so a
microphone left listening looked identical to one switched off — and switching it off looked
exactly like the tap not working, which is how it was first reported. The room's state was
never set either, so the pill read `idle` while she was listening to it. Both are the
interface disagreeing with the microphone.

The first fix for that was itself wrong, and only a person watching could have caught it: the
new pressed style used its own colour, so when she began *answering* the existing
`data-state="listening"` rule stopped applying and the button turned black mid-reply, then
blue again afterwards, for a switch nobody had touched. Two visual languages for one thing.
Cobalt now means *this microphone is open* whatever she is doing with it, and the halo still
animates only while she is listening.

**A room cannot be listening through a face that is not there.** Killing an app does not close
its socket politely, so the server can hold a dead face until a keepalive gives up. The
replacement face arrives, is told `listening: true` because the *room* is open, lights its
microphone button — and is ignored, because its audio is not the open face's. Blue button,
deaf room, nothing in the log, and it was reported in exactly those words. Departure closes it
and a fresh `ready` in a listening room closes it too, so neither has to be the one that works.

**`OnHeard` opened with a bare `if (_responding) return;`.** With a room left listening, that
becomes the most likely way an utterance dies: she answers at length, somebody speaks over the
end of it, and nothing anywhere records that she heard them and let go. Rooms are serialised
and that is correct; being silent about it is not. It is the fourth mute failure found in her
ears in two versions.

### The checks changed with the contract

`'listen' from another room is refused` was guarding the old design and now asserts the new
one — *and* that it still does not open the desk's microphone on the way, because that is the
fault item 9 exists to prevent arriving through the one door item 6 opens. `a tap is not a
press` is new: a tap that took the floor on its way past would spend a quarter-second of her
attention and a line in her log every time somebody flipped a switch.

The embedder press checks now outwait the hold delay, which is a real change in feel worth
stating: **the hold does not engage for 250 ms**, so that a tap and a press can share one
button. Nobody speaks in that quarter second — they are still pressing.

## 0.27.0 — 2026-09-02

**A client's own settings are drawn in her Settings panel now**, above hers, under a heading
the client chooses. The owner's words: *"it should go — if this is an android client show
these settings, if it is the windows client show those."*

On the handset they lived behind a **long press on an invisible corner of the screen**. That
began as a recovery path, and a good one: a wrong address leaves an app whose page cannot
load, and then there is no panel to open — which is exactly where that handset was on
09/01/2026, and it took `adb` to get out of. But a recovery path is a poor place to keep a
setting somebody wants to change, and choosing which camera she looks through is not an
emergency. The long press stays for the case it was built for.

**The page is handed fields and never told what it is embedded in.** `window.OctaviaEmbedder`
gains `name`, `settings()` and `set(key, value)`; `bridge.js` renders whatever comes back —
text, number, password, switch, choice — and hands changes straight back. Nothing in the page
mentions Android, which is the rule the seam was written under in v0.25.0: *the page
special-casing one client is how a renderer stops being a renderer*. A Windows client
describing its hotkey through the same call would get the same rows for free.

A browser tab has no embedder, so the section is absent rather than empty. The credential is a
password field, because it was previously drawn in clear text at full width — not a secret
from the person holding the phone, and very much one from whoever is standing behind them.

**The reply channel grew a value.** `talking` and `watch` only ever needed to succeed or fail;
`settings` has an *answer*, so a reply now carries one when there is one.

## 0.26.2 — 2026-09-02

**Nothing spoken into a phone had ever reached her.** Found while specifying Stage 14 item 6,
which cannot be built on a path that does not work. Three faults in the gap between a button
coming up and the words existing — transcription is asynchronous, and everything here assumed
it was not.

### The transcript was produced and thrown away

`ReleaseFloor` flushes, which hands the samples to Whisper and returns immediately. Then, when
nobody at the desk is listening, the very next line calls `recogniser.Stop()` — which clears
`_wantListening`. A second later transcription finishes, finds the flag down, and returns
without telling anyone.

**So the floor was taken, the audio arrived, Whisper ran, and the log showed a press followed
by nothing.** It worked at the desk because `_wantsToListen` is true there and `Stop` is never
reached; it could never work from a room, which is the only place that path is used.

`_muted` and `_wantListening` guard *incoming* audio. Applying them to words already in flight
was the bug. A flushed utterance is somebody's deliberate act and is delivered whatever the
flags have done since; the voice detector's own segments still respect them, because those
really are the room being overheard.

### The words were attributed to the wrong room

`talking:false` clears `_floor` synchronously, so by the time the transcript existed `EarsRoom`
read the empty floor and answered `host`. A sentence spoken into a phone was judged by the
**desktop's** attention gate and, if it survived that, answered in the desktop's room —
`RespondTo` overwriting the correct `_attending` that `TakeFloor` had just set.

Latched at flush time now, in `_owed`, and read once by whoever the words turn out to belong
to. `Flush` returns whether a transcript is actually coming, so a latch is never left standing
for a press that caught nothing.

### The gate judged push-to-talk, which the code said it did not

> *"push-to-talk bypasses the gate entirely, since a held button has already answered the
> question it asks"*

It did not. Everything the recogniser produces goes through `OnHeard` whatever source it came
from, so a held button was judged like anything overheard — and the gate drops anything under
twelve characters as *"too short to be addressed to anyone"*. Press, say "yes", get ignored.

**`RoomChecks` builds its session with `Gate = "off"`**, so the one harness that drives a face
taking the floor is the one that could not see this.

A press bypasses the gate now, and an asked-for utterance that scores below `MinConfidence` is
logged and kept rather than dropped at debug level — Whisper is least confident exactly where a
phone is worst, a microphone at arm's length, so that filter fired hardest on the face that
could least afford it.

### Four ways to lose an utterance, all of them mute

That is why this survived twenty-six versions. Every silent path says something now: a press
with nothing voiced in it, a press Whisper found no words in, and a low-confidence transcript
that was asked for anyway. *"I pressed the button and nothing happened"* was indistinguishable
from four different failures.

Confirmed from the handset: press, speak, `turn in room 'phone'`. Twice, plus a deliberate
empty press that reported itself. 288 assertions still pass.

## 0.26.1 — 2026-09-02

**Her ears open when the server starts**, and that closes Stage 14 item 11 — the last open
fault in that stage.

The ask was a convenience: *"when the server starts up, the ears should auto start, no point
in that only being activated when its needed."* It turns out to be the clean fix for a logged
bug, which is the second time that has happened. **A fault that needs a mechanism to survive
it is often a fault that should not be reachable.**

### Item 11, closed by removing the window rather than surviving it

The first press of a cold session lost its opening words: Whisper takes a few seconds to load,
a face streams the instant it asks for the floor, and frames arriving before `_floor = from`
were dropped. The two fixes recorded for it — buffer the pressing face's frames, or
acknowledge `talking` — would both have made the loss *survivable*. Loading the models at
startup, where nobody is talking, means there is no gap to survive.

Verified: `talking` on a fresh session takes the floor with no *"her ears were shut; opening
them"* line, and `hello` reports `ears Whisper large-v3-turbo (local)` on the first
connection. What is left is the first two seconds of the server's life.

**It opens no microphone.** Constructing the recogniser builds the voice detector and the
transcriber and touches no device; `Start` is what opens one, and that still needs `listen` or
a held button. `OpenEarsOnStart` is on by default and costs about **1.6 GB resident from
boot**, which is the honest price and is a config key for machines where it is the wrong
trade.

It lives in `Being` rather than the session, deliberately: a server should be *ready* rather
than merely reachable, but that is a hosting decision, and the in-process checks build a real
session dozens of times a run without wanting a 1.6 GB model each time.

### The server never once shut down cleanly, and nothing said so

Found while testing the above: **eight server starts in the log and not a single
"Octavia server stopped"**. `Console.CancelKeyPress` only ever covered Ctrl+C and Ctrl+Break
— and closing the console window, which is now the obvious thing to do since there is a
desktop shortcut that opens one, went down a path nothing handled. Her MCP servers are child
processes and she now holds 1.6 GB of speech model, so an exit that skips `Dispose` leaves
both behind.

`PosixSignalRegistration` covers all three: on Windows .NET maps SIGINT to Ctrl+C, SIGQUIT to
Ctrl+Break and **SIGTERM to the window's close button**, and the same lines mean the right
thing if she ever runs on Linux. The handler holds the signal until the unwind finishes rather
than returning into a race with it.

The proof is the log: *"Octavia server stopped"* now appears, having never appeared before.

### She knows what day it is

Asked the date, she was answering from her training data — months stale and stated with
total confidence, which is the worst combination there is. A server knows what day it is, so
it says so: `Situation` now carries the clock alongside the music, every turn.

Three things about where it went:

- **Not the system prompt**, which is the other obvious home and the wrong one. It is
  identical every turn precisely so it can be cached; a clock in it would break that on every
  question.
- **Not the history.** `Situation` rides with the question and is never written down, which
  matters more for a date than for music — a date kept in the conversation would still be
  claimed as "today" an hour later, and by then it is wrong.
- **`RoomHour` must never reach it.** That pins the room's *lighting* to an hour; the room
  can be told it is evening while she correctly says it is two in the morning.

The persona also gained one line — *"You are told the current date and time with each
question. Use it, and never say you have no way of knowing what day it is"* — because a model
handed the answer will still sometimes recite the disclaimer.

Verified against the system clock: *"Today is Wednesday, 2 September 2026, and the time is
2:07 AM."*

---

## 0.26.0 — 2026-09-02

**Stage 15: she is a server now, and the things that look at her are clients.** One process
became three assemblies — `Octavia.Core` (her), `Octavia.Server.exe` (a headless host) and
`Octavia.exe` (a window). Specced first, in the vault as *Stage 15 — A server, and clients*.

**Nothing on the wire changed.** That was the claim Stage 3 made and this is the bill coming
due on it: the Android client connects to the new server unmodified, and the desktop client
is now a face exactly like it.

### Most of it was already built, twice, under other names

The finding that made this one change set rather than three:

- **The session never knew WPF existed.** Three references to a UI framework in 9,361 lines,
  and only one of them inside `OctaviaSession` — `SaveDiagnosticsAsync`, which had to be
  answered rather than moved.
- **`FaceHub(page, sockets)` already took a nullable page.** A server is that argument being
  null. It was not a refactor, it was a call site.
- **`RoomChecks` had been running her headless for a version already** — a real
  `OctaviaSession` against a recording transport, no window, fifty assertions. That harness
  *is* a server minus a listener.

### What had to be answered

**`saveDiagnostics` could not keep its file dialog.** A dialog needs a dispatcher and
somebody looking at it, and the host may now be running where there is neither — so the one
control that exists for when she is broken would have been broken by moving her. It writes
into `data\diagnostics\` and answers with the path, which is *better*: the machine that needs
diagnosing is usually not the one you are standing at.

**The page lost its second transport.** `postMessage` existed only for the WebView2 page
hosted inside her own process. Left in, `send` would have reported success into a void — and
it was suppressing the *"lost the connection"* notice for exactly the face that now needs one.

**So the page had to learn to reconnect.** While she *was* the window, a dead socket meant a
dead application and there was nothing to reconnect to. A server restarts on its own. There is
a persistent bar now — *lost her — reconnecting* — and a backoff from 500 ms to 15 s, and
`ready` is re-announced on **every** open, because a reconnected face is a new face to the
host: new `FaceId`, no room, no senses, nothing remembered.

**`Hotkey` and `StartMinimised` went to `client.json`.** They always described a key
combination registered with *this* Windows session and whether *this* window starts in the
tray; neither means anything to a server. Carried over from `config.json` on first run, so
nobody loses a hotkey they chose.

### The checks changed more than the code did

Both page-driving suites had been *pretending to be the host over `postMessage`*. With that
channel gone they speak the protocol instead, through a real `WebSocketFaceServer` — which is
a fairer test than the one it replaces, and immediately proved its worth: it is why the
"reconnected faces must re-announce" rule was found rather than shipped.

**`EarsTest -- split` is new** and checks the boundary as *text*: the client never constructs
a session or a brain, the core never reaches for a file dialog or `Application.Current`, the
page has one transport and reconnects, and one version covers all three assemblies. A
compiler cannot express those, because all three assemblies legitimately see each other's
internals. Broken on purpose first — four went red, sixteen came back green.

272 assertions pass, and the renderer conformance check passes against the server.

### Two icons, and the old one was quietly wrong

`tools\make-shortcuts.ps1` writes `Octavia Server.lnk` and `Octavia.lnk`, because there are
two executables now and the server has to be up first.

**It found a fault it was written to prevent.** The existing `Octavia.lnk` had pointed at the
client exe with `--profile dev` since the machine move. After the split that argument reached
a process which does not parse it, and the icon opened a client with nothing to attach to —
no error, no warning, nothing in the log. *An argument that stops being understood is not an
error, it is silence*, and a shortcut is precisely the place nobody re-reads.

The server's console opens normally rather than minimised (`-Minimised` to change it): the
first thing anybody does with a new server is watch it start, and that window is also how you
stop it. It wears her icon, borrowed from the client's assets rather than copied.

### What this deliberately does not do

**The host room still means "the room the server is standing in"**, and while the server runs
on the machine with the devices that is *correct* rather than merely convenient. It breaks
when the box moves to a cupboard — and paying for it in the same change set would have meant
neither half could be tested alone. Open as item 3, along with a portable core (item 2), the
server as a service (item 4), and bundles over HTTP (item 5).

The practical consequence today: **run the client on the server's machine, or use the neural
voice.** Her voice plays through the server's sound card for the host room, and SAPI cannot
be streamed.

---

## 0.25.1 — 2026-09-01

**A room face could never start her ears**, so the microphone button restored yesterday could
not work until somebody walked to the desk and pressed a different button. Reported from the
handset against v0.25.0, with the evidence on a live socket: `micAccepted: False`,
`ears: not started`, `listening: False`.

**This is item 9's bug, not item 10's**, and it had been sitting there since item 9 landed —
invisible because until v0.25.0 no remote face had a microphone button to press.

### `listen` was doing two jobs

- **Opening this machine's microphone.** A host-room device. Item 9 made that host-only and
  that is exactly right.
- **Starting the speech recogniser.** *Being-wide* — it is the same Whisper for every room,
  and a face taking the floor is an explicit request to be transcribed.

Item 9 correctly locked the first and, without anybody noticing, took the second with it.
`TakeFloor` required a running recogniser, only `listen` started one, and `listen` was now
refused from the only rooms that needed it.

They pull apart cleanly. `EnsureEarsAsync` opens and wires the recogniser exactly once,
whoever asked; `StartListening` then adds the host's own microphone and the room-music
analyser on top. **Holding the button opens her ears**, and the first press on a cold session
may take a moment while a model loads — letting go during that takes nothing, which is why
there is now a `_pressing` separate from `_floor`.

### `micAccepted` meant the wrong thing

It reported whether the recogniser was *already open*, which is false on every fresh session.
So a handset was told its microphone button could only fail, hid it correctly, and the fault
looked like a client bug. It now means **will accept** — the recogniser is Whisper, or is not
open yet and would be.

### Three things that would have gone wrong quietly

**`UseSource` starts what it is given.** So the obvious line in `ReleaseFloor` — hand the ears
back the local microphone — would have **opened the host's microphone because a phone let go
of a button**. Exactly the fault item 9 exists to prevent, arriving through a door item 9 did
not fit a lock to. When the desk is not listening the recogniser is stopped instead, and there
is a check that fails if the desk's microphone opens during a remote press.

**`Start` subscribed without detaching**, so a face taking the floor on ears the desk already
had running would have processed every frame twice. One `-=` before the `+=`.

**A mute outlives a turn.** `Unmute` only ever ran when the *desk* was listening, so after her
first reply the ears would have been open, the source right, and nothing heard. Taking the
floor now unmutes — a held button is an explicit request to be heard.

### Checked

Three new assertions in `EarsTest -- rooms`, and one of them really opens Whisper: the model
is already on this machine, so a room face pressing the button on a session where nothing has
ever listened is driven for real, and skipped with a note on a machine without it. Reverting
both halves of the fix turns the first two red.

Fifty assertions there now. The suite passes end to end.

### Confirmed from the handset, and one thing left

*Added after release, 09/01/2026.* Verified from the phone: `micAccepted: True` on a fresh
session, and holding the button opened her ears without anybody touching the desk — the
placard went from `EARS not started` to `EARS Whisper large-v3-turbo`.

**The first press of a cold session still loses its opening words.** Opening Whisper takes
about 3 seconds with `large-v3-turbo` on CPU, and the client starts streaming the moment it
sends `talking(true)` because nothing acknowledges the floor — so every frame before
`_floor = from` is dropped at `OctaviaSession.cs:121`.

It is worse than the failure beside it. Letting go mid-load yields **silence**, which is
legible. This yields a **plausible, truncated sentence**: press, speak immediately, and the
opening words are gone with nothing to say so — she answers the wrong question and everything
looks fine.

Recorded as ROADMAP item 11 and deliberately not fixed in this release. The likely shape is
buffering frames from `_pressing` until `_faceMic` exists: no protocol change, no client edit,
and it hides latency that is real rather than adding any. Only the first press is affected —
a warm press reaches `_floor = from` without ever awaiting.

---

## 0.25.0 — 2026-09-01

**A renderer can borrow the senses of whatever it is embedded in.** Stage 14 item 10, built
from `Stage 14 - Lending A Renderer The Device's Senses.md`, written on the Android side.

The ask: *"I want us to somehow put back the microphone button on the phone and wire it in
correctly. I don't like the key button press — the UI should feel the same on the phone and
the host. Also, if I switch on the camera will her face and eyes follow me?"* Two requests,
one seam.

### What item 9 left behind

Two controls hidden on a face outside the host room, both for good reasons:

- The **microphone**, because it sends `listen`, which drives the *host machine's*
  microphone — the whole point of the authority table.
- The **watch button**, because `navigator.mediaDevices` does not exist outside a secure
  context, so on a plain `http://<lan-ip>` origin it could only throw. That was 0.24.1.

Both correct, and together they left a handset **less capable than the machine it stands in
for** — on a device that has a microphone and a camera, and whose native client already owns
both. The way in was a hardware volume key, which works and is not what anybody wants to look
at.

### Why neither could be fixed on the wire

**The microphone.** The floor is a `FaceId`:

```csharp
if (_floor == from) _faceMic?.Push(pcm, pcm.Length);
```

The face that presses must be the face that streams. The Android app is **two** faces — a
native connection that owns the microphone and a WebView panel that draws her page — so a
button in the panel taking the floor would have the host dropping the native client's audio
as coming from somebody who does not hold it. Making the floor room-scoped instead would let
*any* face in a room feed her ears, which is a worse rule than the one there is.

**The camera.** Watching is renderer-local by design and `PROTOCOL.md` is emphatic about it:
nothing — no frame, no coordinate, no flag — crosses the socket. That is right and is not
touched here. The problem was only that "the page" is the wrong place to get the pixels when
the page is embedded in something that has a camera and its own origin is not secure.

### The seam

```js
window.OctaviaEmbedder = {
  senses: ['mic', 'camera'],
  talking(held) {},
  watch(on) {},
};
```

Five changes in `bridge.js`, **no protocol change**, and a browser tab behaves exactly as it
did. It is deliberately not an Android interface — the page special-casing one client is how
a renderer stops being a renderer.

Two decisions worth writing down:

**A borrowed camera is not claimed to the host.** `senses` in `ready` still reports only what
*this page* can do. The embedder lends gaze, not stills, so a panel that claimed a camera
would be sent a `look` it cannot answer — and on a handset would take that frame away from
the native client, which can. It looks like an oversight and is the opposite; there is a
check that goes red if anyone "fixes" it.

**The privacy marker stays in the page.** One marker, in the place a person already looks. An
embedder drawing its own would be two things to trust rather than one.

### The one place the interface cannot match the host

The desktop's microphone button is a **toggle**: `listen` opens her microphone and leaves it
open, and the attention gate decides what was addressed to her. A remote room cannot have that
yet — an open microphone beside a speaker playing her voice, across a network with latency
each way, is the echo problem item 6 deferred, and Android's `AcousticEchoCanceler` is
per-device and not dependable.

So: **same button, same place, same look, held rather than toggled.** That is the whole of the
difference and it is said in the code rather than hidden. Making it a toggle needs real echo
cancellation and item 7's per-room gate actually built; if the difference is unacceptable the
answer is not a smarter button, it is doing item 6 properly.

### Every way a press can end

A held button that never releases holds her ears until the host's sixty-second floor timeout,
so `pointerup`, `pointerleave`, `pointercancel`, `blur`, `visibilitychange` **and the socket
closing** all release, and the release is idempotent. Dragging off cancels, which is what a person expects; the
system taking the gesture — a scroll, a call arriving — cancels; backgrounding the app
cancels.

The socket one is worth saying separately, because it is the least obvious and the worst: the
audio goes out over the **embedder's** connection, not this page's, so a panel that loses its
own socket would keep streaming from a button whose release it can no longer be trusted to
report. It is also the one release path with no check behind it — the harness has no socket —
and that is said here rather than left to be discovered.

### Checked in the real engine, because all of it lives in the page

`EarsTest -- embedder` drives the actual page in WebView2 and injects the embedder with
`AddScriptToExecuteOnDocumentCreated`, which is where a WebView host puts it. Twenty-one
assertions across three faces: a handset with an embedder, a plain browser on the LAN with
none, and the desktop in the host room.

**The two origins are reproduced rather than simulated.** `https://octavia.face` is a secure
context and `http://octavia.face` is not, so `getUserMedia` is genuinely absent on the second
exactly as it is on a handset — and the first assertion checks *which one it got*, so a run
that passes because the simulation broke says so instead.

Four mechanisms broken on purpose to watch the right checks go red:

| Broken | What went red |
|---|---|
| the borrowed-microphone gate | *a borrowed microphone brings the button back* |
| `pointerleave` / `pointercancel` removed | both release checks |
| a borrowed camera claimed in `senses` | *a borrowed camera is not claimed to the host* |
| the borrowed-camera watch gate | *a borrowed camera brings the watch button back* |

Two bugs in the harness itself, both worth the note: `AddScriptToExecuteOnDocumentCreated`
**accumulates**, so without removing the previous script the handset's embedder would have
been injected into the "plain browser" run and the two checks proving a browser tab is left
alone would have passed while testing the wrong page entirely. And `ExecuteScriptAsync`
returns the completion value, which is not always a string — `window.__calls = []` evaluates
to `[]` and threw halfway through, reported as the checks failing to start.

### Also

`PROTOCOL.md` gains the embedder contract — marked *not part of this protocol*, because
nothing here crosses the socket — and a note that **a still and a watch want the same
camera**. `look` can arrive while a watcher runs; a renderer that binds the device
exclusively will kill the watcher mid-gaze and leave her staring at the last place she saw
somebody. The built-in page has the mild version of this today.

---

## 0.24.1 — 2026-09-01

**A button offered where it could only throw, and a blocker that never existed.** Both found
from the Android side driving v0.24.0 from a real handset.

### The watch button asked the wrong question

`bridge.js` gated it on the host's setting alone:

```js
watchBtn.hidden = !msg.camera;
```

`msg.camera` is the **room's** answer to "may she look at all". It is not this renderer's
answer to "could I, physically" — and on a plain `http://<lan-ip>` origin
`navigator.mediaDevices` is undefined, because a browser only offers cameras to a secure
context. So a remote face got a control whose only possible outcome was
`face error: watch failed: Cannot read properties of undefined (reading 'getUserMedia')`.

The test was already sitting nine lines up the same file, computed in v0.24.0 for `senses`
and then not reused. It is named now — `canOpenACamera` — and the button asks both questions.

**The setting itself stays visible on such a face**, deliberately: it belongs to the *room*,
and in that room the native client is the one that answers `look`. Watching is renderer-local
— the motion centroid never crosses the socket — so it is the only one of the two that needs
this page to own a camera.

The camera picker's empty-list hint was wrong in the same way, telling somebody on a phone to
"allow it once" and come back, which sends them looking for a permission prompt that can
never appear. It now names the real reason.

### There was never a missing API key

Stage 14 item 1 recorded, in v0.21.0:

> two real cameras were never opened, because `MaybeLookAsync` requires the Claude brain and
> there is no key on this machine

The first clause is true and is the whole answer. **The second was never true.**
`data\apikey.dat` decrypts under this account to a 108-character `sk-ant-…`, and the `cloud`
profile sets `Brain: claude`; `home` — the default — sets `Brain: local`, which is why `look`
never fired. It needed a profile, not a secret.

It traces to a note from the 08/30 machine move saying the key had been left behind and
needed re-pasting. That was true when written. It was re-pasted the same day and the note was
never updated, and the stale sentence then propagated into **three** roadmap entries: the
"there is a camera now" note, item 1's landed note, and — inherited in good faith — item 9's
reason for holding criterion 7 open.

All three are struck. So is the tool loop's second excuse in stage 12, which read "there is no
API key on this machine to verify it against"; that one deferred real work for two versions on
a false premise. **Its first reason still stands** — the loop changes the path every turn
takes — but it can now be written and watched to run.

### Criterion 7, closed on hardware

Driven from the Android side against v0.24.0. A probe face joined room `phone` declaring
`senses: []`, leaving the native client as the only camera in the room:

```
look: asking face a85b541d in room 'phone'
sight: 1280x960, brightness 0.57, spread 0.190
look: got a frame, 97 KB
```

and in the handset's own log, `CameraStill: one frame, 97 KB`. The WebView panel — `9e649919`
— was never asked, which is precisely what `senses` was added for, and the alternative was a
coin flip. She described the frame correctly. `setCamera` behaved per-room throughout:
`camera=false` on connect, `true` after asking, one `warn` line naming the room.

**Stage 14 item 9 is fully closed**, all ten criteria, and item 1's carried-forward gap with
it.

### The lesson, which is one this project has already paid for once

Yesterday a vault note claiming the camera "has not been opened" was corrected because the
config disproved it, and the lesson written down was *a note that records a setting records a
moment, not a fact*. This is the same failure one layer up: a note recorded a **transient
task** — "the key needs re-pasting" — and it hardened into a permanent property of the
machine. Nobody re-tested it, because it read as a finding rather than as a to-do.

A claim about the environment should carry the command that proves it. "There is no key" is
unfalsifiable prose; `--profile cloud` starting on `brain: claude-sonnet-5` is a check.

**Verified here rather than inherited in the other direction**, which is the same mistake
pointing the other way: she was started on `--profile cloud`, the log says
`brain: claude-sonnet-5`, and the brain pill is **green** — that dot is `hasKey`, so the key
decrypts under this account rather than merely existing as a file. No call was made.

### A thing the log gave up for free

Reading the day's camera lines to caption the screenshot showed the v0.24.0 per-room rule
working, unprompted and unaided:

```
12:10:40  warn   camera enabled in room 'phone'
12:13:24  info   camera disabled in room 'phone'
12:47:45  warn   camera enabled in room 'phone'
14:02:15  warn   camera enabled in room 'host'
```

Every `phone` line lived in memory and died with the process; only the `host` one reached
`config.json`. Naming the room in that warn line cost four words and turned "is the camera on,
and who turned it on" into something answerable by reading.

It also surfaced that **the camera is currently enabled at the desk** — deliberately, at
14:02, and left rather than quietly switched off here. Worth turning off when nobody is
testing.

---

## 0.24.0 — 2026-09-01

**Two rooms.** Built from `Stage 14 - Two Rooms.md`, a specification written on the Android
side by the client that needed it, for whoever was working in her repo. It supersedes item 5
(turn ownership), absorbs item 7 (the attention gate), and struck item 4 as already done.

The ask, in the owner's words: *"On the phone I should not be able to toggle the host
mic/keyboard. Say one day I am at the gym and accidentally click it and no one is at home.
One brain, same avatar, same personality, but different spaces."*

Two separate faults, kept apart because one is five lines and the other is the architecture.

### 1. A phone could drive this machine

`listen` toggles **the host's microphone**. The handset renders the same page as the desktop,
so the mic button at the gym opened a microphone in an empty house. `PROTOCOL.md` was honest
that this is deliberately not a room-level control — and **nothing enforced it**: no `set*`
case in `OctaviaSession` looked at `inbound.From` at all. The same was true of
`setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder` and
`saveDiagnostics`.

There is an authority table now, checked on the room a message came from, before the switch
acts:

| Class | Rule |
|---|---|
| **Host only** — `listen`, `setMicrophone`, `setOutput`, `setMusic`, `setWhisperCompute`, `openDataFolder`, `saveDiagnostics` | Only from the host room. Elsewhere: refused, answered with a `notice`, logged once per room per kind. |
| **Room** — `say`, `talking`, `hush`, `forget`, `sight`, `setCamera`, `setCameraDevice`, `selfTest`, `faceError` | The sending face's room only. |
| **Being** — `setKey`, `setVoice`, `setVoiceEngine`, `setAvatar`, `setRoomHour`, `setStats` | Any room, echoed to every room. Every face is wearing the result. |

`hello` gained `controls`, and the page hides its host-only rows when it says `room`.
**That is a hint and not the enforcement**, and both are needed: without the guard a face can
send the message by hand anyway, and without the hint a phone shows a microphone button that
silently does nothing, which is its own kind of broken.

Refusing rather than ignoring matters too. A face that quietly does nothing looks broken, and
somebody will spend an evening on it.

### 2. There was one conversation, and every face was a window onto it

`RespondTo` did not take a face. `caption`, `turn`, `state`, `emotion` and `cleared` all went
out with no target, which means everyone. `_brain` held the one `Conversation` there was. So
typing at her on the phone put your words on the desktop's screen and played her answer in a
room you were not in.

A **room** now owns what makes a conversation a conversation — its history, its state, its
mood, its floor, and whether a camera may open in it. The being still owns everything that
makes her *her*.

**`Conversation` is lifted out of `IBrain`.** There are N conversations and one of her, and
`Forget()` used to clear the only one there was. Constructing a `ClaudeBrain` per room would
have duplicated the HTTP client and the key in order to keep two lists of strings apart;
`RespondAsync` takes the history instead, which `Conversation.cs` was already shaped for.

**Her voice was the one that was actively wrong** rather than merely coarse. `SendAudio`
reached every face that had opted in, so she spoke aloud in rooms nobody was standing in. It
takes a room now — and this machine's speakers are silenced for the length of a turn she is
having somewhere else. Silenced at the sound card and nowhere earlier: the visemes and the
streamed PCM are read from the same buffer at the same instant, so a phone still gets her
voice in step with her mouth, and only the room she is not in goes quiet.

The Windows voice cannot be streamed at all, so a remote room is *told* — a `notice` saying
she can be read and not heard — rather than her falling back to talking out loud at an empty
desk.

**`_lastSpokenThrough` is gone, as its own comment asked.** A face declares `senses` in
`ready`, and `look` goes to one that claims a camera, in the room that asked. It matters
concretely on Android: the native client owns the camera and the WebView panel cannot open one
at all, because `getUserMedia` needs a secure context and the panel is served over plain HTTP
— so without `senses` the host had a coin-flip chance of asking the half of the phone that
physically cannot answer.

**An absent `senses` is not an empty one.** A face that predates the field is a candidate of
last resort rather than a refusal, which is what keeps `attach-face.ps1`, the checks and the
built-in page working with no changes at all.

### The decision that kept it small

**Rooms are serialised.** She attends one at a time. One `_responding` flag, one `_turn`, and
the other room is refused out loud with *"She is talking to someone else."* — exactly what
`TakeFloor` already did for the floor, generalised from *the floor* to *her attention*.

Making it re-entrant "since there are rooms now" is the concurrency change this defers, and it
is also untrue to the thing being modelled: one being cannot hold two conversations at once,
and pretending otherwise is a worse simulation rather than a better one.

### Nothing existing changes

A face that names no room is in the host room. The built-in page needed no edit to keep
working, and neither did `attach-face.ps1` or `EarsTest`. The room is **not** derived from the
credential, tempting as that was — token-means-loopback-means-host would put two handsets in
one room and make a laptop on the LAN indistinguishable from a phone.

### Checked, and broken on purpose first

`EarsTest -- rooms` drives a real `OctaviaSession` through a recording transport and a
forty-line stub of a local model, and asserts all ten of the spec's acceptance criteria
in-process — no handset, no API key. Forty-seven assertions.

Each of the four mechanisms was then broken deliberately, to watch the right checks go red:

| Broken | What went red |
|---|---|
| `ToRoom` broadcasts again | six, including *nothing of it reaches the desktop's placard* |
| the authority table removed | eleven — and the phone's `setMusic` really did change the host's settings |
| one `Conversation` shared by every room | *another room starts from nothing* |
| her voice always aloud | *the log says where her voice went* |

The conversations in the check run between two **non-host** rooms on purpose: her voice is
silenced in any room but the host's, so the suite proves the routing without the machine
talking out loud on every run.

~~**Criterion 7 is the exception and is said out loud.** `look` needs the Claude brain and
there is no key on this machine, so the *choice of face* is asserted directly against the rule
and the end-to-end half is still owed — the same gap item 1 recorded honestly rather than
counting as done.~~

**Closed the same day, and the reason given here was false** — see 0.24.1. There was a key all
along; `home` is simply a local brain. The round trip walked from a handset on `--profile
cloud`, and the in-process assertion of the *choice of face* stands beside it.

**Criterion 8 checked itself.** Three seconds after the first build with rooms in it started, a
real handset at `10.1.1.181` authorised with the remote key and both of its connections — the
native client, which asked for audio and skipped visemes, and its WebView panel — landed in
room `phone` together. The Android side had been sending `ready.room` against the spec before
the host understood the field; until this build it was ignored.

Two things went wrong in the checks themselves, and both were in the test rather than the
code. The stub model only understood `Content-Length`, and `PostAsJsonAsync` sends chunked —
then, having learned to read chunks, it left the trailing CRLF in the receive buffer, which
makes Windows close with an RST and the client discard a response it had already been handed.
It presents exactly as a model server that is not there. And the helper that waited for a turn
treated *any* notice as the end of one, so "her Windows voice cannot leave this machine" ended
the wait before she had said a word.

### Still owed

- **A Settings row that displays the remote key**, unchanged from 0.23.1. `EarsTest --
  remotekey show` is still the only way to read it.
- ~~**Criterion 7 end to end**, which needs a key.~~ Closed in 0.24.1, from a handset. It
  never needed a key; it needed `--profile cloud`.
- **The Windows Firewall being off entirely.** Unrelated to this, and still worth undoing in
  favour of a scoped inbound LAN rule.

---

## 0.23.1 — 2026-09-01

**The remote key could never let anybody in.** One character, latent since 0.14.0, found
the first night a phone came in over WiFi instead of `adb reverse`.

`RemoteKey.Value` returned a stored key only if it was **24 characters or more**.
`Regenerate()` makes four groups of five joined by three dashes — **23**. The guard could
therefore never pass, so every read of `Value` discarded the saved key and minted a new
one. A remote face offered its key, `Matches` read `Value`, `Value` replaced the secret,
and the comparison ran against a key that had existed for a microsecond. It failed every
time, and the file changed on every attempt.

Nine versions of nobody noticing, because **nothing ever exercised it**: the phone reached
her over `adb reverse`, which is loopback, and loopback uses the per-run token. The key
path had never once been walked until the handset moved to WiFi.

**The fix is not 24 → 23.** That is the same bug with a different number, and it comes
back the next time the grouping changes. The number was hand-computed from a format
defined elsewhere, so the two halves could drift apart in silence. Now `Groups`,
`GroupSize` and `Chars = Groups * GroupSize` are declared once and the generator and the
guard both derive from them, and the guard measures the **normalised** string — dashes,
spaces and case are presentation, so the length test and the comparison can no longer
disagree about which characters are the secret.

The floor is a *minimum*, not an exact shape. A key typed in by hand is a legitimate thing
to find in that file — it is plain text on purpose — and rejecting one would overwrite it
silently, which unpairs every device without ever saying so.

### The check that would have caught it, and the one that proves the fix

`RemoteKeyChecks`: **mint a key, read it back, assert it is the same string.** Nine of its
assertions fail against 0.23.0. It is deliberately the dullest possible test, and it is the
one that mattered — the code was covered by nothing because nothing had ever compared a key
against *itself*.

`Value` is re-read from disk rather than cached, partly so a hand-edited key takes effect at
once and partly so that check is testing the file instead of a field that would agree with
itself no matter what was written.

Proven by breaking it on purpose: setting `Groups = 5` leaves the round trip and the
authentication **green** — the original failure cannot return — while the two shape
assertions go red, so changing the format stays a deliberate act.

### Also

- **One log line per admitted address, not fifteen.** `Authorised()` runs on every request,
  and since the assets went behind the same gate a single page load wrote a dozen identical
  `remote face authorised from …` lines. A wall tablet that reloads would have buried
  everything else. Deduped per address, per run — after a restart, a device getting in is
  news again. **Successes only:** the first cut of this deduped refusals too, which was
  wrong. A hundred bad keys from one address is a different event from one bad key, and
  collapsing them discards the only signal that says so. Repeated failure is the thing
  worth being noisy about.
- `OCTAVIA_REMOTE_KEY` joins `OCTAVIA_CONFIG` and `OCTAVIA_LOG`, so a check can mint and
  re-read a key without unpairing every device that trusts the real one.
- `EarsTest remotekey show` / `roll`. Nothing in Settings displays the key, despite a source
  comment claiming it does, so reading the secret a phone must be told meant opening
  `data\remote.key` by hand.

## 0.23.0 — 2026-09-01

**A microphone somewhere else.** MINOR: a refactor of how she hears, not plumbing. Stage 14
item 2, from [[Stage 14 - A Microphone Somewhere Else]] — the last thing between a tablet
and a peer.

`WhisperRecognizer` did not *consume* microphone audio; it **was** the microphone. It built
its own `WaveIn`, chose the device, and raised into its own frame loop, so there was nothing
to hand a different source to. Now there is `IAudioSource`, with `LocalMicSource` (today's
capture, lifted out unchanged) and `FaceAudioSource` (fed by binary frames from one face).
Everything from the 512-sample frames inward was already source-agnostic and is untouched.

**Push-to-talk, and it is load-bearing.** A held button has already answered *"was that
addressed to me?"*, so the attention gate does not apply to this path and item 7 is not
needed for it. One talker at a time also means one Whisper, so the earlier worry about
sizing two concurrent transcriptions against eight cores does not arise. `talking:false` is
an exact end-of-utterance marker, which `Flush()` acts on directly rather than waiting 800 ms
for the detector to guess.

The floor is held by one face, released by the button, by disconnection or by a timeout —
a phone in a pocket must not own her ears for ever. A second face pressing is refused with a
`notice` rather than silence. Pressing while she is speaking hushes her, because that is what
talking over someone means.

### The two traps the spec warned about, and they were both real

**The room-music analyser is fed from the microphone.** `whisper.Audio += _roomMusic.Push`
was exactly right while there was one microphone in the world. Swap the source wholesale and
that subscription quietly follows the phone — she would report the tablet's kitchen radio as
the music around *her*, and **everything would appear to work**. So the local microphone is
owned by the session, shared, and framed *separately* for music by a small extracted
`PcmFramer`. It keeps running while a face holds the floor: speech moves rooms, her sense of
what is playing here does not.

Which also means `UseSource` detaches the old source but must **not stop** it — the same
trap wearing a different hat, and the thing most likely to be undone later by someone
tidying up. So it is a test: two spy sources, switch between them, assert the first is still
running. Reintroducing `_source.Stop()` turns it red, which was checked rather than assumed.

**`WatchForSilence` would cry wolf.** The deaf-microphone warning names Remote Desktop audio
settings, and it is right to. For a push-to-talk face, silence is the *normal* state —
nobody is holding the button — so every remote session would raise it at somebody holding a
phone. Gated on `ExpectsContinuousAudio`, which is why that flag is on the interface.

### Also

`micAccepted` in `hello`, so a client does not offer a microphone button that could only
fail. Binary frames upstream mirror the downstream rule already in `PROTOCOL.md`: a binary
frame is audio and nothing else, in both directions. Only the floor-holder's frames are
read; anyone else's are dropped rather than mixed.

*Not done, and worth being straight about: acceptance criteria 1, 5, 6 and 9 need a real
handset — a held button producing a transcript, the local microphone genuinely muted during a
remote utterance, the room analyser still hearing this room, and barge-in. What is covered
here is the seam and both traps at unit level, and criterion 6's mechanism directly. The rest
is the first press on the Android side.*

---

## 0.22.0 — 2026-08-31

**Her voice can leave this machine.** MINOR: a new stream on the wire, and every send now
goes through a queue. Stage 14 item 3, built from [[Stage 14 - Her Voice On Another Face]].

### The premise of the spec was wrong, and it matters

The spec opens with a *"do this first"*: `Broadcast` fires `_ = SendAsync(...)` un-awaited
per face, `WebSocket.SendAsync` forbids concurrent sends on one socket, so a burst throws
`InvalidOperationException` — which the catch-all swallows, silently dropping a live face.

**That does not happen on .NET 10.** Measured directly, because a bug worth this much work
is worth reproducing first: 320 overlapping sends on one socket, from many threads, with a
reader draining — **nothing thrown, socket still open**. `ManagedWebSocket` serialises sends
behind a lock rather than rejecting them. A test written against the stated failure passed
against the *unfixed* code, twice, which is what prompted checking the premise at all.

**The real fault is next door, and the queue fixes it anyway.** A face that stops reading
fills its socket buffer; every un-awaited send then stops completing and they accumulate
without bound, holding their buffers alive. A slow phone becomes the host's memory leak.
Proven the other way round — a probe with 40 sends and no reader blocked indefinitely.
Ordering across un-awaited calls is not guaranteed either.

So: **one writer per face, fed by two queues.** Control is unbounded and never dropped —
losing a `state` leaves a face permanently wrong about her rather than briefly behind. Audio
is bounded at 16 frames and drops the *oldest*, because a face that cannot keep up should
hear a gap and catch up. And the catch-all is split: a face that genuinely went away stays
quiet, anything else is logged once. That distinction is the difference between diagnosable
and not.

### The voice itself

- **`IVoice.Audio`** raises the PCM at the moment it reaches the sound card — the one point
  where being in step with the visemes is guaranteed, since both are read from the same
  buffer. The span is copied into a pooled array and returned immediately; a handler may
  not hold it.
- **Binary frames are audio and nothing else.** No header, no tag. A second binary stream
  would be a version decision.
- **Opt-in, uniquely.** `subscribe` gains `want: ["audio"]`, and no face gets audio until it
  asks. `skip` declines a *rendering hint*; audio is a *physical output*, and opt-out would
  mean every browser face on this machine playing her voice over the speakers she is already
  using — her talking over herself in the same room.
- **The format is announced**: `audioAvailable`, `audioRate`, `audioBits`, `audioChannels`
  in `hello`, read from the live voice model. `audioAvailable` is **false** on the Windows
  voice, which cannot be teed — a face is *told* it will get nothing rather than waiting in
  silence, which is the difference between a limitation and a bug.
- **Stopping needs no message.** A non-`speaking` state means throw the buffer away, and the
  host drains its own queues at the same moment. Confirmed rather than assumed: `Hush()`
  does raise `Finished`.

### Also, from the Android side

The test harness starts a **real** `WebSocketFaceServer` on an ephemeral port, into her real
log — so "grep the last `face socket listening` line" could hand back a dead test token for a
port that no longer exists, and the failure presented as *the client* being broken. Servers
now carry a `Label`; the harness's says `TEST face socket`. Cost real time on the client
side, and the fix is one string.

*And a flake I exposed rather than created: `FaceProtocolChecks` asserted `FaceCount == 2`
on the line after `ConnectAsync`, but a client has the 101 response strictly before the
server finishes building the face. Two channel allocations in the `Face` constructor widened
that window enough to lose it every time. It now waits, which it always should have.*

---

## 0.21.2 — 2026-08-31

**The camera gets a switch.** PATCH, but it closes a hole that had been open since Stage 9.

Reported as *"what happened to the camera icon on the UI?"* — and nothing had. The eye
button and the `camera` pill were still in `index.html`, hidden by one line:

```js
watchBtn.hidden = !msg.camera;     // bridge.js
```

`hello.camera` comes from `_config.Camera`, which is `false` by default. Correct behaviour —
a control for a switched-off sense should not sit there — but **there was no camera control
anywhere in Settings**, so the only way to turn her camera on was to hand-edit `config.json`
and restart her.

Half-built rather than deliberately omitted, and the halves were the wrong way round: the
config keys existed, the host handled `setCameraDevice`, `hello` carried `cameraDevice`, the
face *received* it into `wantedCamera` — and the face never sent it, because nothing could.
`camera.js` even exported `cameras()`, commented *"for the settings menu"*, for a settings
menu that was never written. A wire with no switch on either end.

Now: a **Let her see you** checkbox and a **Camera** picker, both in Settings beside the
microphone. `setCamera` is new on the wire, so `PROTOCOL.md` moves — unlike 0.21.0, this one
is visible to another face.

Three details worth keeping:

- **The camera list comes from the face, not the host.** Unlike the microphone and the
  output, which are the host's devices. `camera.js` enumerates them and matches by *label*,
  because a device id is regenerated per origin and per permission grant and cannot be stored
  in a config file and still mean anything tomorrow.
- **A browser withholds device labels until the permission has been granted once**, so the
  picker reads *"Not known yet"* on a fresh machine and the hint says why. The difference
  between a menu that is waiting and a menu that looks broken.
- **Enabling is logged at `warn`.** A camera coming on in someone's home should leave a mark
  that is easy to find later; switching it off is a plain note.

*Caught before it shipped: the first version of the picker's change handler called
`useCamera()`, which is destructured inside `look()` and does not exist at that scope — a
`ReferenceError` on every camera change. It was also redundant, since `look()` applies
`wantedCamera` on each look, and calling it there would have broken the promise in
`camera.js` that a face never asked to look loads no camera code at all. The `SyntaxChecks`
harness would **not** have caught this: it parses and checks `ready`, and a ReferenceError in
an event listener is neither.*

Verified both directions against a live browser face: off → checkbox clear, picker disabled,
eye button hidden; on → `"Camera": true` written to `config.json`, picker enabled, eye button
back in the console, `warn camera enabled from settings` in the log. The test config was
returned to `false`, which is where it started.

---

## 0.21.1 — 2026-08-31

**She has a face on the outside of the window too.** PATCH: branding, no behaviour.

Until now she had **no icon at all** — no `<ApplicationIcon>` in the csproj, so the exe wore
the generic .NET box, and the tray fell back to `SystemIcons.Application`. From a supplied
mock sheet she now has an exe and window icon, a tray icon, and a favicon on the page the
socket serves, so a phone that shortcuts to her face gets her mark rather than a globe.

`tools\make-icons.ps1` cuts the panels out of the sheet and writes the whole set. It is
committed rather than scratch: there is no transparent master yet, and when one arrives the
answer should be one command rather than an afternoon of re-deriving crops.

Two decisions in it worth keeping:

- **The mask is a shape, not a colour key.** Keying black to transparent would punch holes
  straight through artwork that is mostly dark. Clipping to the rounded square and the circle
  the design already draws removes the surround and leaves the art alone.
- **Nothing is upscaled.** Every size is a downscale from its panel. A 512px web icon was
  generated and then deleted — the source panel is 315px, so 512 would have been invented
  detail presented as resolution.

**The tray icon does not really work, and that was measured rather than assumed.** Rendered
at a true 16×16 and magnified, the colour circle is an unreadable purple blob and the
monochrome version is worse — a dark smudge, near-invisible on a dark taskbar. The circular
mask rescued it as far as it goes: it is now a clean purple *disc* rather than a black square,
and it reads as a distinct dot. That is what shipped, deliberately, with the reason written
down. A circuit-brain, a face profile and a HUD ring cannot survive 256 total pixels; the
concentric ring at her temple could, and is the obvious tray-only variant if it is ever worth
doing.

The vault gains [[Branding]], holding the **source sheet** rather than only the generated
icons — an `.ico` cannot be re-cropped, re-masked or exported at a size nobody has asked for
yet, so the thing the icons were cut from is the thing worth keeping. Dated, so a later sheet
sits beside this one instead of replacing it.

---

## 0.21.0 — 2026-08-31

**Faces have identity, so `look` stops opening every camera in the house.** MINOR: an
architectural interface changed and a user-visible behaviour changed with it. Stage 14
item 1, implemented from a specification written on the Android side — see
[[Stage 14 - Face Identity]].

`IFaceTransport` was broadcast-only in **both** directions: `Send` went to everyone and
`MessageReceived` said nothing about who sent it. That was a deliberate simplification and
exactly right while every face only watched. It stopped being right the moment a face had
a camera.

**The bug it fixes.** `look` was broadcast and the session waited on a single promise. With
a tablet and the desktop both attached, one question needing eyes **opened both cameras**,
lit the privacy marker in an empty room, and whichever frame arrived first won —
arbitrarily. That quietly broke the promise `camera.js` makes in its own header: *it is
never opened unasked.*

- `FaceId` and `FaceMessage` — the id was always minted per connection in
  `WebSocketFaceServer`; it simply never left the class. Letting it out is most of the work.
- `Send(object message, FaceId? to = null)`, where **null still means everyone**. That
  default is why the change stayed small, and it is the same instinct as `subscribe` being
  opt-*out*: a new message type reaches every face rather than being withheld from the ones
  nobody remembered to address. `SendTo` still honours a face's `skip` list — being
  addressed directly is not a reason to override what it asked for.
- `_looking` carries the face it asked, and a `sight` from anyone else is **logged and
  dropped** rather than silently accepted.
- The `look` target is the last face a *person* spoke through — `say`, `listen`, `hush` —
  falling back to the built-in page when the utterance came from the PC's own microphone.
  **This is not turn ownership**, it is one field, and the comment says so: proper ownership
  is item 5 and this must be replaced by it rather than grown into it.

Nothing here is visible on the wire, so `PROTOCOL.md` does not move — which is the sign
this is a seam being repaired rather than an interface being widened. The diff is six
files, all host-side; `wwwroot` is untouched.

**Verified against the spec's acceptance list**, with one honest gap:

| | |
|---|---|
| A targeted send reaches one face, the other hears nothing | `EarsTest`, two live sockets |
| A `sight` from an unasked face is ignored and logged | **Live**, two browser faces |
| `caption`, `turn`, `state` still reach every face | **Live** |
| `subscribe`/`skip` still applies per connection | **Live** — visemes and levels skipped, the rest arrived |
| The built-in page works with the socket unbound | **Live** — port held by another process, `face ready` still arrived |
| `FaceStatus` and the diagnostics count unchanged | Untouched in the diff |

**The gap: two real cameras were never opened.** `MaybeLookAsync` requires the Claude brain
and ~~there is no key on this machine~~, so the camera path cannot run end to end here. The
routing is proven at the transport level and by the `sight` guard firing live; the camera
itself is not. ~~Worth re-checking on a machine with a key before trusting it completely.~~

> **Closed 09/01/2026 in 0.24.1, and the reason above was false.** There was a key the whole
> time — the default `home` profile is simply a local brain, which is the first half of that
> sentence and all of the explanation. This is the sentence that cost four versions: item 9
> inherited it in good faith as the reason its own criterion 7 could not be walked, and
> nobody re-tested the claim because it read as a finding rather than as a to-do.

*The silence assertion — "the face that was not addressed received nothing" — first killed
the very face it was proving had been left alone. **Cancelling a `ReceiveAsync` aborts a
WebSocket, it does not time out**, and that lesson was already written down in this project.
Racing the read against a delay is the version that works.*

---

## 0.20.1 — 2026-08-31

**The contract now says what the code does.** No behaviour change; documentation only, which
is exactly why it is a PATCH and exactly why it mattered enough to do straight away.

`PROTOCOL.md` had fallen behind `OctaviaSession` by **five face→host messages** —
`setMicrophone`, `setOutput`, `setCameraDevice`, `setWhisperCompute` and `setStats` — and by
**eight `hello` fields**: `cameraDevice`, `stats`, `microphones[]`, `microphone`, `outputs[]`,
`output`, `whisperCompute` and `toolServers[]`.

This stopped being cosmetic the moment a second face existed. An Android client is being
written against that document by someone who cannot read `OctaviaSession`, and the gap was
found the hard way — `say` carries `text` while every `set*` message carries `value`, and the
first implementation guessed `value`. She accepts the wrong field and silently does nothing
with it: no error, no log line, a dead Send button. **A contract that is quietly incomplete
is worse than one that is honestly small.**

Also corrected two stale strings the vault tooling had been printing since the repo went to
GitHub: `_Code Index` claimed she was "not yet on GitHub", and `check-vault.ps1`'s own
documentation said the snapshot was the only off-machine copy. Both have been untrue since
08/30/2026.

## 0.20.0 — 2026-08-31

**Her face is now reachable from another machine.** MINOR: the socket learned to serve
HTTP, which is a new subsystem and the thing Stage 13 was actually blocked on.

### The problem, which was not the one written down

Stage 13 step 1 opened the *socket* to the LAN in 0.14.0 and everyone — this changelog
included — recorded that as "a transport that can leave the machine". It was half of one.
`wwwroot` never reached the built-in face over a network at all: it arrives through
`CoreWebView2.SetVirtualHostNameToFolderMapping`, which is a **WebView2 feature, not a
server**. Nothing in this process had ever answered a GET. A phone has no equivalent of a
virtual host, so it could open the socket and still have no page to run in it.

The alternative was vendoring `wwwroot` into the client, and it lost on a fact rather than
a preference: **there are two virtual host mappings, not one.** The second serves her
avatars folder, and `AvatarUrl()` hands the face `https://octavia.avatar/<file>`. A VRM is
*user data* — in a git-ignored folder, chosen at runtime, in no repository — so it can
never be baked into a client, and the host would have had to serve files regardless. The
choice was one mechanism or two, and two of them drifting.

### What landed

- **`StaticFiles`** resolves a request path to a file under `wwwroot` (`/`) or her avatars
  folder (`/avatars/`), and says no by default: traversal is checked on the *resolved*
  path rather than by hunting for `..`, and an extension not on the list is refused rather
  than sent as octet-stream. It is not a file share.
- **`WebSocketFaceServer` answers GETs** that are not upgrades. The listener was already
  parsing request lines for the handshake, so this is a branch rather than a second server
  — and it means the page and the socket share one origin and one port.
- **A cookie, because sub-resources cannot carry a query string.** `<link href="face.css">`
  and `import('./watch.js')` are resolved by the browser, which knows nothing about a key;
  gating them on `?key=` would have served the page and then refused everything in it. The
  credential is echoed back `HttpOnly`, `SameSite=Strict`, and the assets present that.
- **`bridge.js` stops hardcoding `127.0.0.1`.** A face served over HTTP came *from* the
  socket, so it addresses the socket as the origin it was loaded from. The built-in page is
  not served by the socket and still gets told with `?port=`, so it is unchanged. It also
  accepts `?key=` now, which is what a remote face presents.
- **The avatar URL is rewritten in the renderer**, not in `hello` — one `hello` is
  serialised once and broadcast to every attached face, so the host cannot say something
  different to each. The face is the only party that knows which origin it is on. This is
  the first place "faces have no identity" (Stage 14) actually bites.

### Proven, not reasoned about

Loaded in a real browser over HTTP while her own WebView2 face was attached — two faces at
once, which is what Stage 3 was for. Splash green on all three steps, every module and the
vendored three.js 200, the 14.4 MB VRM fetched from `/avatars/`, eighteen `blob:` texture
loads, and her character on screen with a face. Traversal (plain and percent-encoded), the
`/avatars/../config.json` escape, and an unserved file type all 404; an asset with no
credential is 401.

The one console error is `ERR_NAME_NOT_RESOLVED` with no matching network request, and it
reproduces on a bare stylesheet with no scripts, socket or avatar — so it belongs to the
browser harness, not to her.

**Still to do before a phone can use this:** remote access is off by default and needs a
Windows firewall rule scoped to the LAN subnet, and `getUserMedia` will not run on a plain
`http://10.1.1.x` origin — which is why the Android client owns the camera and microphone
natively and leaves the WebView as a renderer.

## 0.19.3 — 2026-08-31

**Four roadmap bugs closed, one of which was live.** MINOR-adjacent but PATCH: nothing
changes shape, but her default brain does.

**She was one shortcut away from being mute.** The config on this machine said
`Profile: "live"` → `Brain: "claude"`, and no key has ever been stored — so every turn
would have hit "No API key yet. Paste one below and try again." She worked only because
the shortcut passes `--profile dev`. Started any other way, from the exe, from
`dotnet run`, from a fresh shortcut, she could not answer at all. The API-key nag that
was cluttering her face in 0.18.x was this, correctly reporting a real misconfiguration.

Stage 10's local-first profiles are now built, and the important half is not the new
names: **base `Brain` defaults to `local`**. An unnamed or misspelled profile falls back
to the base settings, and a keyless Claude is the worst possible thing to fall back to —
she looks broken rather than limited. Shipped profiles are `home` (local,
`large-v3-turbo`, the default), `cloud` (Claude) and `dev` (local, `small.en`). `live`
still resolves to what it always meant, so no existing config changes behaviour. An
undefined profile is now a **warning** that names the ones that exist; it was an info
line, which is exactly how this hid.

**Stage 10a is closed, both halves.** `EarsTest` gained `SyntaxChecks`, which takes the
option that entry preferred: load the real page in the WebView2 already on the dependency
list and let Chromium parse it. No hand-rolled parser, no Node, same engine that runs it
for real, and served from the same virtual `https://` origin so the CSP behaves as it does
in the app. It asserts no `SyntaxError`, `window.Face` published, and that the bridge would
have sent `ready`.

It was proved by breaking it. An orphan `});` appended to `bridge.js` — the v0.18.0 fault
exactly — turns it red and names the file and line, with `window.Face` still fine and
`ready` missing, which is that bug's precise signature. The host half, `WatchForFace`,
gives the face 30 seconds to say `ready` and otherwise logs an error and shows the
fallback panel: a surface that survives the renderer being dead, which is the whole point.

**Stage 2a is closed.** `LocalBrain`'s streaming loop reads to null instead of testing
`EndOfStream`, so it no longer blocks a thread pool thread waiting for the token it then
politely awaits. The build is at **zero warnings**. The audit that entry asked for came
back clean: `EndOfStream` occurred exactly once in the project.

**Docs.** `README.md` described a console that has not existed since 0.18.0 and opened by
calling her "a talking bust with Claude behind her" — both halves of that are now wrong.
It documents the drawer and its tabs, the typing button, the status-readout setting and
the local-first default. `PROTOCOL.md` turned out to already carry `camera` in both the
`hello` table and its own `look`/`sight` section, so that item was closed before it was
opened.

*A check nobody has watched fail is not a check. The syntax check was written, passed
immediately, and that proved nothing — the only informative run was the one against a
deliberately broken file.*

**Late addition — the visual record gets a tool and a rule.** The screenshot folder had
gone quiet at v0.15.0, and its two most recent files had never been written up at all.
`tools\shoot.ps1` captures her real window at the 1100x780 every existing shot uses, taking
the version from `Octavia.App.csproj` so a shot cannot be filed under the wrong release;
`tools\poke.ps1` clicks and scrolls in window coordinates, which is what you measure off the
previous shot, so a whole set can be **retaken** after the chrome moves. Both verify the
window actually came forward before acting, because `SetForegroundWindow` fails silently
from a background process and the capture then photographs the wrong window.

`check-vault.ps1` now reports when the current version has no shot. It reports rather than
fails: most releases change no pixels, and the point is to make somebody look.

Six shots taken for v0.19.3 — the first set that includes her **window** rather than only
her face. One of them carries the two UI fixes from earlier today in a single frame, and
another is simply the proof this release existed: `PROFILE home`, `BRAIN qwen2.5:7b-cpu
(local)`. The **v0.10.0–v0.11.0 console rebuild is a permanent gap** — that console is gone
and cannot be photographed now, which is the argument for the rule.

---

## 0.19.2 — 2026-08-31

**The typing field stays where you left it.** PATCH.

Sending a line closed the field. That was written as "the field has done its job and the
room comes back", which is a reasonable sentence and wrong about how anybody actually uses
it: a person who typed once is usually about to type again, and they were made to re-open
it for every single line. `submitTyped` no longer closes anything — it clears the box and
puts the cursor back in it.

Closing is now the keyboard button's job, or **a minute of not using it**, whichever comes
first. The timer restarts on every keystroke and every send, and it only ever fires on an
**empty** field: a half-written line is somebody still thinking, and a strip of chrome is
not worth throwing their sentence away. Escape still closes it immediately, unless she is
talking, in which case Escape stops her instead — that was already true and is unchanged.

**Roadmap gains stage 2a**: `LocalBrain.cs:96` is `while (!reader.EndOfStream)`, which is a
*synchronous* read — the CA2024 warning, and the only one in the build. On a server-sent-event
stream there is nothing to peek at until the model emits the next token, so the loop blocks a
thread pool thread waiting for the line it then politely `await`s on the next statement. It
costs more here than it used to: the brain is pinned to the CPU, tokens are slow, and the
thread is parked for the length of every reply.

*Tested by temporarily dropping the timer to 1.5s and driving the three cases — empty field
closes, draft survives, post-send field closes on its own clock — then restoring the minute.
A one-minute timeout that is only ever reasoned about is a one-minute timeout nobody has run.*

---

## 0.19.1 — 2026-08-31

**The placard fades instead of collapsing.** PATCH: fixes the rescale that came with it.

v0.19.0 gave the placard a `max-height` collapse specifically so it would not leave a band
of empty room behind. It worked, and it introduced a worse problem: the placard was a flex
sibling *below* the stage, so collapsing it grew the stage by 92px. The canvas is watched by
a `ResizeObserver`, a resize recomputes the fit and the camera distance, and the result was
that she visibly jumped size every time the caption came and went. The thing meant to give
her the room was re-framing her twice a cycle.

The fix is to stop the stage changing height at all. `#placard` now lives **inside** `#stage`
as an absolutely-positioned overlay — a subtitle over the room rather than a strip beneath
it — sitting on a short gradient scrim so the text stays legible against a bright wall, and
`pointer-events:none` so it never takes a click meant for her. The room is full height in
both states; only opacity animates.

Measured both ways: with the caption up and down, `#stage` and `#scene` both report 615px,
so the observer never fires and the camera never moves.

*The hidden-pane artifact from 0.19.0 bit again and is worth restating in its sharper form:
while the pane is hidden, `requestAnimationFrame` never fires and running transitions freeze.
A frozen transition reports a settled-looking `opacity` — 0.65 in this case — that is neither
end of the animation, and it reads exactly like a CSS rule being overridden. Test the cascade
by setting `transition:none` and reading the two states directly; that answer does not depend
on anything being rendered.*

---

## 0.19.0 — 2026-08-31

**She gets the room when nobody is talking.** MINOR: the placard comes and goes.

- **"In residence" is gone** from the header. It meant "she runs here rather than in the
  cloud" — true, unremarkable, and not worth a permanent line in the room.
- **The placard collapses after nine seconds of quiet** and the stage takes the space, so
  she is simply bigger when nothing is being said. Any caption or any state but idle brings
  it straight back.

Two details that took a moment to get right. The linger is **nine seconds, not zero**: a
reply that vanishes the instant she stops speaking is unreadable, and the caption is the
only record outside the transcript. And it **collapses rather than fades** — `opacity:0`
alone leaves a 92px band of nothing, which is the opposite of the point. `min-height:0` is
not enough either, because the caption still has an intrinsic height; it takes `max-height`
against a real number to animate down to nothing.

**Roadmap gains stage 10a**: nothing in this project checks the face's own syntax. A
JavaScript parse error is invisible to `dotnet build` and to `EarsTest`, so the build goes
green and the face is simply dead — which is how v0.18.0 shipped a broken bridge for a few
minutes. The entry proposes parsing `wwwroot/*.js` in the harness, and the more valuable
half: having the host notice that `ready` never arrived and say so.

*Two hours were lost to a measurement artifact worth writing down: **the browser pane does
not run WebGL or style recalculation while it is hidden**, so screenshots came back with an
empty scene and `getComputedStyle` returned stale values. Both looked exactly like real
bugs. Front the pane and capture in the same batch before believing either.*

---

## 0.18.0 — 2026-08-31

**The room gets quieter.** MINOR: the status readout moved and became optional.

- **The drawer button sits outboard of the state**, to the right of it. The state is about
  the room and reads first; the drawer is the way out.
- **The status readout floats over her top-left corner** on translucent glass, instead of
  taking a strip of chrome along the bottom. It is reference material — glanced at, not
  read — so it belongs *in* the room quietly rather than in a band of its own. It does not
  take clicks: a readout that swallows a click over the scene is a trap.
- **`ShowStats`** turns it off entirely, from Settings. On by default, because it answers
  most of the questions anyone asks about her; off is the setting for actually looking at
  her.
- **The missing-key pill is gone.** A permanent amber warning across the bottom of the
  room is one you stop seeing, and it nagged whoever was looking at her rather than whoever
  could act on it. Settings carries it now: the field is marked, its label reads **API key
  — needed**, and it gains an amber rule down its edge so someone who came for something
  else still finds it.

*(A `sed` deleting four lines by number took the wrong four and left an orphan `});`,
which broke the whole bridge — the drawer stopped opening. Caught by the console, not by
the build, because nothing compiles this file.)*

---

## 0.17.0 — 2026-08-31

**The console gets its corners back.** MINOR: the control surface moved.

Three changes, from a marked-up screenshot:

- **The drawer button is in the header**, beside the state it belongs with. It had been
  sitting in the console row among the microphone and the keyboard — a settings menu
  filed with the controls you reach for mid-sentence. Smaller up there, because it is a
  way out rather than something you press while talking.
- **The status readout moved to the right of the control row**, from underneath it. The
  left of that bar is where you act and the right is where you look; having both stacked
  down the left made it read as a wall of text with a microphone on top.
- **The row spans the full width** instead of a centred 860px column, which was pulling
  both ends toward the middle and leaving the corners empty.

The pills are right-*positioned* but left-*aligned* inside their block. Right-aligning the
text as well gave every label a different starting x, and a list you scan down wants one
edge to run your eye along.

**Not fixed: the headphones.** Four attempts, and adjusting by eye did not converge — each
round fixed the last complaint and introduced another. Height is right; width and depth
are not. What was learned is written into ROADMAP.md stage 11 so the next attempt starts
from it: `Box3.setFromObject` on a skinned mesh returns the *bind pose* (the body reads
±0.69 wide, arms out), the face mesh is half-width 0.109 against the 0.164 in use, and
measuring the skull alone buries the cups in a long-haired character's fringe. The missing
quantity is the hair silhouette at ear height. Worth a dev-panel slider rather than a
rebuild per guess.

---

## 0.16.2 — 2026-08-31

**The duplicate music logging, which was not what it looked like.** PATCH.

Five identical `music: N bpm` lines in one second looked like the two sources talking over
each other. It was neither source: the transition was logged on `state.Playing !=
_last.Playing`, but `_last` is only assigned **after** the 80 ms throttle returns early —
so the same change re-reported on every frame until a send finally went through and moved
`_last` on. The log now happens where `_last` is updated, so the two cannot drift apart.

Both sources do also need telling apart, so `MusicWatcher` has a `Name` — `output` for the
loopback, `room` for the microphone — and it appears in the line.

**And they were genuinely competing.** Both wrote to the same face state with no rule, so
the tempo would flicker between what this machine plays and what is in the room, with
neither reading trustworthy. The loopback now wins whenever it has something: clean
dynamics, no room, no gain control. The microphone only speaks when the loopback is silent.

---

## 0.16.1 — 2026-08-31

**The cups sit on her ears, and the room source is proven.** PATCH.

### Room music, verified

v0.16.0 shipped `MusicFromRoom` untested against real room audio. It works: with music
playing on a *different computer* and the loopback probe reading **energy 0.00**, she
reported **141 bpm at confidence 0.49** through the microphone. Both halves of that matter
— the silent loopback is what proves it is the mic, and 0.49 against loopback's 1.00 is
exactly the reduced certainty a boom mic in a room was predicted to give.

### The headphone depth, measured rather than argued

Height was right after v0.16.0; depth was not, and the sign was the reason. `vrm-avatar.js`
notes that VRM 1.0 faces -Z, which invites the assumption that "behind the head" is +Z.
**After the loader plugin has finished, it is not**: the head bone's axes come out
world-aligned and the eye bones sit at *greater* z than the head bone, so the face points
+Z and behind is negative.

Settled by shoving the whole assembly to +0.10 and watching it land in front of her nose —
one deliberately wrong value being worth more than another round of reasoning. The offset
is now 0.26 of the head half-width behind the eye line, which puts the cup on the ear
rather than the temple or the back of the skull. The comment records the measurement so
the next person does not re-derive it from a misleading premise.

---

## 0.16.0 — 2026-08-31

**She can hear a room, her headphones sit on her ears, and the dance stopped twitching.**
MINOR: a new source of hearing.

### Stage 11a — the microphone as a second ear for music

Loopback is what *this computer* plays. A speaker across the room — another PC, a phone —
never touched it, which is how "she will not dance" turned out to mean "the music was on a
different machine". `MusicFromRoom` runs a second `MusicAnalyzer` on the microphone frames
the voice detector is already reading, so it costs one subscription and no extra capture.

Off by default, and honest about two things: a boom mic in a reverberant room gives far
worse dynamics than a loopback, so expect less certainty; and it only works while her ears
are open, which is a different condition from `Music` and will surprise someone.

**Built and passing every check, but not yet verified against real room audio** — that
needs a speaker in the room, which is a thing to do rather than a thing to assert.

### The headphones sit where her ears are

They were sized and placed from head *height*, an assumption about width taken from an
unrelated measurement. They now come from the **eye bones**, which VRM 1.0 requires: the
ear canal sits at eye height and slightly behind, so that is two measurements instead of
two guesses. Cups widened so they sit outside the hair rather than buried in it — with
thanks for the marked-up screenshot, which showed the gap immediately.

### The dance was being driven by two staircases

`sway` came from `sin(beats * PI/2)` where `beats` was an **integer counter** — a step
function — and the beat impulse decayed a tenth per frame. Eight bones driven off those is
exactly what "jittery" looked like.

There is a continuous **phase** now, in beats, advancing at the detected tempo; each beat
only eases it a quarter of the way toward the nearest whole beat rather than driving
anything. Sway is a smooth function of phase over a bar, the per-beat bounce is a raised
cosine, and nothing anywhere is a spike. It keeps ticking when the music stops so she eases
out mid-stride instead of freezing on the last value.

### A latent bug that would have hit the first good monitor

**The canvas was laid out at the size of its drawing buffer.** `setSize(w, h, false)`
deliberately leaves the CSS alone, `#scene` relied on `inset:0`, and a canvas is a
*replaced* element — with width and height auto it takes its intrinsic size, which `inset`
cannot stretch. On any display with `devicePixelRatio > 1` the canvas therefore overflowed
its stage by exactly that ratio, anchored top-left, and she rendered enormous and off to
the bottom right.

Invisible here because WebView2 runs at dpr 1 on this machine, and waiting for the first
4K monitor or a laptop with display scaling. Found only because the browser pane emulates
dpr 2 — and confirmed pre-existing by reproducing it on the previous commit before
assuming it was today's work.

---

## 0.15.2 — 2026-08-31

**"She will not dance" is almost never the analyser.** PATCH.

Reported: music playing, no dancing, and a reasonable guess that her music sense was off by
default. It is not — `Music` defaults to true and the log showed the loopback open the
whole time. **The music was playing on a different computer**, and WASAPI loopback taps
what *this* machine plays. She was listening perfectly to an output nobody was using.

Nothing was broken, so the fix is to make the invisible thing visible:

- The Music self-test names the endpoint she actually **opened** — not
  `LoopbackListener.DefaultDevice()`, which is a different question now that `OutputDevice`
  exists — and its fix line says to check the music is coming out of that one.
- `MusicSummary` names the device even while she *is* hearing something, so the log answers
  the question before it is asked.
- The Settings hint names it too: *"Listening to Headset Earphone. Play music through that
  one — she hears a single output, not the machine."*

Verified end to end afterwards: 183 bpm at confidence 1.00 from the probe, then she wore
her headphones and danced, tracking 110 → 138 bpm as the track changed.

**Also found, and left as [[Roadmap]] stage 11a:** she cannot hear a room at all. Loopback
is what the *computer* plays; a speaker in the same room only reaches the microphone, which
is not wired to the analyser. The mic read 0.013 against a 0.004 floor, so it could hear it.
For something that lives in a room rather than on a desktop, that is the wrong way round.

*(A label/`for` double-toggle was suspected in the music checkbox and tested: one click, one
toggle, one event. Not the bug. Recorded so nobody suspects it twice.)*

---

## 0.15.1 — 2026-08-31

**The microphone was never broken.** PATCH.

`EarsTest mic` judged its reading against a single threshold, so a quiet room and a dead
device both came back `SILENT` — and it spent a morning looking like a fault. It reports
the endpoint's level and mute state now, and gives the same three verdicts the self-test
does: signal, room noise, or genuine digital silence. The Jabra reads 100%, unmuted, and a
noise floor of 0.004 that rises when spoken to. Working all along.

The same single-threshold mistake in the self-test was fixed in v0.12.0; this is the other
half of it, in the tool someone reaches for first.

---

## 0.15.0 — 2026-08-31

**The gate and the brain stop being the same model.** MINOR: a notable behaviour change,
and a documented rule reversed.

### A constraint that was the VM's, not the design's

`GateModel` had to equal `LocalModel`, and the self-test **failed** when it did not. The
reason was real and measured: on the 16 GB dev VM the server could hold one model, so a
separate gate meant a swap on every utterance — 24 seconds against 0.7 for a warm call.

Re-measured on 32 GB: **a 3B gate and a 7B brain both stay resident**, alternating calls
run at 0.39 s and 3.1 s with no swap at all, and 15.9 GB is still free. The check is a note
now rather than a failure, and the config recommends the split — the two jobs want opposite
things, and no one model was best at both.

### Bigger was not better at anything measured

New probe: `EarsTest models <name>...` times a warm gate judgement, scores it against four
mixed cases, times a spoken reply, and counts how many of four unambiguous requests produce
the right tool call. CPU-only, which is what this machine has:

| model | gate | correct | reply | tools |
|---|---|---|---|---|
| `llama3.2:3b` | 688 ms | **4/4** | 4.9 s | 4/4 |
| `qwen2.5:3b` | **308 ms** | 2/4 | **1.6 s** | 4/4 |
| `qwen2.5:7b` | 815 ms | 2/4 | 5.1 s | 4/4 |
| `llama3.1:8b` | 1843 ms | 2/4 | 9.5 s | 4/4 |

Two things worth keeping:

- **Every model called tools correctly**, including the 3B. The assumption that small
  models cannot be trusted with Stage 12 was wrong on these cases — though four
  unambiguous requests is a floor, not a hard test.
- **Both Qwens answer NO to "what is the weather doing tomorrow".** They fail closed on a
  gate that is supposed to fail open, and a gate that never opens is worse than none. It
  looks like prompt sensitivity rather than capability, and it is worth chasing:
  `qwen2.5:3b` at 308 ms would be the better gate if the instruction can be made to land.

**Configured:** gate `llama3.2:3b-cpu`, brain `qwen2.5:7b-cpu`. Gate median **665 ms** on
the split, and the brain is now a model that can hold a conversation for the same reply
time the 3B took.

### Answers folded into the roadmap

- **Google Home, no Home Assistant.** Google publishes mobile SDKs and a manufacturer API;
  there is no supported way for a Windows service to control Google Home. HA is therefore
  the recommendation *to make the stage possible*, not a preference — and most of those
  devices are Matter or WiFi underneath, which HA can hold alongside Google without
  removing anything.
- **UniFi UDM SE at `10.1.1.1`**, folded in through HA's UniFi integration rather than a
  second server.
- **The UDM SE's own WireGuard is enough; Tailscale is not needed** unless the ISP is
  CGNAT. Forwarding WireGuard's UDP port is a different proposition from forwarding hers —
  a WireGuard endpoint is silent to unauthenticated packets. The Windows firewall still
  needs one inbound rule, scoped to the LAN.

---

## 0.14.0 — 2026-08-31

**A door she can open to a phone, and a body that moves when the music does.** MINOR: a
new transport mode and a changed performance.

### Stage 13 — the groundwork, which is not the app

Steps 1–4 of the stage, all of which live in this repo. The Android client does not, and
is not pretended to.

- **`RemoteAccess`** binds every interface as well as loopback. Off by default, logged
  loudly when on, and the single riskiest line in the project — which is why it is a
  switch rather than a default.
- **`remote.key`** — a durable, readable, groupable secret in her data folder. The per-run
  token is regenerated every start, which is right for a page the host loads a second
  later and useless for a phone in a pocket. **The per-run token is not accepted from off
  the machine at all**: it is written to the log and carried in a URL. Regenerating the
  key revokes every device at once, which is the whole revocation story.
- **`subscribe`** — a face may name message types it does not want. Opt-*out*, so a face
  that never sends it keeps getting everything and no existing renderer changes. A phone
  sends `{"skip":["viseme","level"]}`, because sixty visemes a second is a battery rather
  than a feature.
- **The network decision, written down** in PROTOCOL.md: Tailscale or Wireguard, never a
  forwarded port. One shared secret in front of a microphone and a house is enough behind
  a tunnel and is not enough on the open internet.

### She dances with more than her head

Head-only movement reads as listening. A person moving to music moves from the hips, and
the shoulders and arms follow a beat late. `setDance(amount, sway, beat)` joins the avatar
interface: hips, spine, chest and arms, every bone written as rest + offset so the idle
pose is not lost the moment the music starts, and the chest counter-rotating against the
hips because a torso does that and a board does not. Small angles throughout — the line
between dancing and convulsing on an anime rig is about fifteen degrees. The bust ignores
it, having no body to move.

### Audit

- **A race in the code written an hour earlier.** The per-connection `skip` set was a
  `HashSet` written on one connection's receive thread and read by `Broadcast` from
  whichever thread produced a viseme. It is an immutable set swapped by reference now:
  replacing a reference is atomic, editing a set under a concurrent read is not.
- **`WasapiCapture` is obsolete** in this NAudio, and the peak probe was the only thing
  still using it — two capture paths in one codebase is how a diagnostic ends up
  disagreeing with the thing it diagnoses. It is `WasapiRecorderBuilder` now, the same as
  the loopback, sharing the same `AudioSamples` decoder. `Peak` became `PeakAsync`.
- The probe now says when **no buffers arrived at all**, which is the one thing a peak of
  zero cannot tell you on its own: a quiet room and a dead device read identically
  otherwise.
- Build is clean: **no errors and no warnings**.

---

## 0.13.0 — 2026-08-31

**A splash, a room with air in it, and the beginnings of hands.** MINOR: a new subsystem
and a changed console.

### Stage 11 — the interface

- **A loading splash**, held until the scene has built *and* the host has answered. She
  used to show a finished-looking console while the renderer, the socket and the voice
  were all still coming up, and the gap between looking ready and being ready is where
  every "she ignored me" report starts. It names the step it is on, shows a startup notice
  in place — a voice or a speech model downloading is the thing that actually takes the
  time — and opens anyway after fifteen seconds rather than stranding anyone behind it.
- **Typing costs a click.** The text field was the width of the window in a console where
  most turns are spoken. It is behind a keyboard button now, opens focused, closes on send
  or Escape. Hush moved out of the field, because she can be speaking while it is shut.
- **The status strip stacks bottom-left.** Five readings on one line read as a sentence to
  parse; five short lines read as a list to scan, and scanning is all that happens from a
  sofa.
- **The headphones are placed rather than guessed.** They were sized from the character's
  *height* (`headPoint.y * 0.115`) — an assumption about head width taken from an unrelated
  measurement, wrong by a different amount for every model — and added at the head bone's
  origin, which is the base of the skull, so the band hung at the neck. Both are measured
  now: the head bone against the top of the model's own bounding box.
- **The room has air.** 260 additive motes drifting in the key light, their opacity
  following the key so they vanish at night rather than reading as fog; the near parallax
  slab leans on the beat; and the camera sways a few centimetres on a very long period.
  That last one is what makes the rest work — parallax is relative motion, and against a
  bolted-down camera the layers slide while the scene still reads flat.

### Stage 12 — the tool seam

**The seam is built and tested; she cannot call a tool yet.** Both halves of that are
deliberate. See ROADMAP.md.

- `ITool` / `IToolProvider` / `ToolRegistry`, and `McpClient` speaking **MCP over stdio** —
  handshake, `tools/list`, `tools/call`, newline framing, per-request timeouts, and a read
  loop that fails pending calls when a server dies instead of hanging a turn.
- **`ToolRisk`, and the rule that dangerous tools do not run unasked.** MCP carries no risk
  annotation, so it is inferred and biased towards asking.
- `McpServers` in config; tokens belong in `Env`, not in arguments.
- `tools\mock-mcp.ps1`, a three-tool server, so the seam is testable with no house
  attached. 11 new checks drive the real client against a real child process.

The brain-side tool loop is **not** written: it changes the working conversation path and
~~there is no API key here to verify it against~~. Writing it blind and calling it done would
repeat precisely the mistake v0.12.0 spent its length undoing.

> **The second reason was never true** *(struck 09/01/2026, see 0.24.1)*. There is a key and
> `--profile cloud` reaches Claude. The first reason stands on its own and is the real one;
> this one deferred work on a false premise for two versions.

### Stage 13 — away

Designed, not built, and the design says the app is the last step rather than the first.
The prerequisites — a transport that may leave the machine, an auth secret that survives a
restart, the Tailscale-or-Wireguard decision, and a protocol subset a phone would want —
all live in this repo. The client itself is a separate project needing an Android SDK and
a device.

---

## 0.12.0 — 2026-08-31

**Four things were broken, and three of them had been blamed on something else.** MINOR:
her face, her ears' placement and her hearing of music all change behaviour.

### Her textures never loaded, and it was the CSP

`img-src` allowed `'self' data: https://octavia.avatar`, and **`blob:` was not among
them**. glTF keeps its textures inside the binary; three.js decodes them into a `Blob` and
loads them from a `blob:` URL — which that policy blocked, for every texture, in every
model, of every format. `connect-src` lacked it too, so the ImageBitmap route failed the
same way. Proven in the browser before changing anything: `IMG BLOCKED`, `FETCH BLOCKED`,
`BITMAP BLOCKED`, then all three passing after adding `blob:` to both lists.

She has 20 textured materials of 28 now, at up to 2048×2048, with zero texture errors —
against `hasMap: false` and eight failures before.

**The KTX2 theory in ROADMAP.md was wrong**, and so was the lighting explanation that
preceded it. `vrm-avatar.js` still never calls `setKTX2Loader`, which remains a real
latent gap for a model that needs it, but it was not this.

### The microphone check called a working headset silent

It read `AudioMeterInformation.MasterPeakValue`, on the reasoning that Windows' own meter
shows signal without opening a capture. **That meter reports zero unless something already
holds the device open**, so an idle machine always measured exactly 0.000. It opens a real
WASAPI capture now and reads the Jabra's true noise floor of 0.004, and reports three
states rather than two — speech, room noise, and genuine digital silence, which is the
only one that is a fault.

### The music path was decoding the wrong bytes

A crest factor of 1.7 had been read over Remote Desktop, through a virtual streaming
endpoint and through two different real sound cards, and blamed on each in turn. It was
none of them.

A shared-mode mix format is `WAVE_FORMAT_EXTENSIBLE`, not `IeeeFloat`, so the float test
failed and the decoder fell through to `ToInt16` — **taking the low two bytes of each
32-bit sample**. Those bits are uniform noise, whose RMS is 0.577 and whose crest factor
is 1.73. That is exactly what was measured, to three decimals, on every device.

Fixed by testing the sub-format GUID and decoding 8/16/24/32-bit properly, in a shared
`AudioSamples` so the diagnostics cannot drift from the capture. Against a played 132 bpm
track: crest **1.7 → 7.7**, peak 0.793 and RMS 0.103 matching the source exactly, and the
tempo **131.8 bpm at confidence 1.00** where it used to wander between 75 and 184.

`EarsTest music demo` now compares what it captured against the crest factor of the track
it played, instead of against an absolute threshold that assumed the signal was fine.

### She was thinking on a 2014 graphics card

`ollama ps` said `4%/96% CPU/GPU`: 28 of 29 layers offloaded to a **GeForce GT 730 over
Vulkan**. Ollama does not need CUDA, so being Kepler did not save it. Every attention-gate
call took ~3.9 s against an 8 s timeout, and the gate probe's median was **8009 ms** —
pegged at the timeout, failing open on all eighteen lines. She was answering the
television, and the only symptom anyone could see was that she felt slow.

The OpenAI-compatible endpoint ignores `options.num_gpu`, so placement is pinned with a
Modelfile instead — `PARAMETER num_gpu 0`, created as `llama3.2:3b-cpu`. Gate median
**8009 ms → 640 ms**; the corpus 144.2 s → 16.5 s.

- **New `Gate speed` self-test** times a warm call and names this cause, because nothing
  in the config was wrong and no amount of reading it would have found this.

### Settings that stop this happening silently again

- **`MicrophoneDevice`, `OutputDevice`, `CameraDevice`** — pick a device instead of
  inheriting the Windows default. Matched by substring, which is required: `WaveIn`
  truncates names to 31 characters, so the same headset is two different strings.
  Dropdowns in Settings, populated by the host over `hello`.
- **`WhisperCompute`** (`auto`/`cpu`/`gpu`) and **`WhisperThreads`**. Note that
  Whisper.net's own default order is CUDA-first, so "auto" is not neutral. On this
  machine CUDA never loaded at all — the GT 730 is below CUDA 12's floor — so Whisper was
  always on the CPU. Measured with `small.en`: 4 threads 5.55 s, 8 threads 4.12 s,
  16 threads 3.66 s.
- `EarsTest compute <auto|cpu|gpu> [model] [threads]` measures it rather than assuming.

### Also

- `attach-face.ps1` read the log from `%APPDATA%`, which v0.11.0 moved. It now resolves
  the data folder the same three ways `Paths.cs` does.
- `hello` reports the devices and the compute choice; `attach-face` prints them.

---

## 0.11.0 — 2026-08-30

**She keeps her things where she lives.** MINOR: every path she writes to moved.

Her data folder was `%APPDATA%\Octavia` unconditionally. On the move to the physical PC
that folder was left behind by a copy that took the repo and the vault, and its contents —
Whisper models, Piper voices, and **the only `.vrm` avatar** — were lost. The avatar was
unrecoverable: no note in the repo or the vault had ever recorded which model it was.

`Core\Paths.cs` now resolves the data folder in three steps: `OCTAVIA_DATA` if set;
otherwise `<repo>\data` whenever she is launched from a build inside the repo, found by
walking up from the executable for `Octavia.slnx`; otherwise `%APPDATA%\Octavia`. So
`dotnet run`, a Debug shortcut and `dist\Octavia.exe` all agree, copying the project now
copies her models and her face with it, and an installed copy under Program Files still
writes somewhere it is allowed to.

Everything routes through `Paths`, so this was one file and 38 call sites needed no
change — the seam was already there.

- `data\` is git-ignored and excluded from the vault snapshot: hundreds of megabytes of
  downloaded artefacts, none of it source.
- Existing data moved from `%APPDATA%\Octavia` into `<repo>\data`; the old folder is
  parked as `Octavia.old` rather than deleted.
- Two replacement avatars added — `AvatarSample_A.vrm` (VRoid, VRM 0.x) and
  `VRM1_Constraint_Twist_Sample.vrm` (pixiv, VRM 1.0). Neither uses KTX2, so neither
  reproduces the texture fault; see ROADMAP.md.
- Docs follow: `<data>` is defined once in README.md and in the vault's
  [[Profiles & Configuration]], and the literal paths elsewhere now point at it. The
  historical record in this file and the Changelog is left as it was written.

---

## 0.10.0 — 2026-08-30

**Stage 10 — the console rebuilt, and her face made legible.** MINOR: the whole control
surface changed shape.

**Partially delivered, deliberately versioned anyway.** The work was interrupted before
its follow-up items, and a snapshot whose `<Version>` said 0.9.2 while carrying an
entirely different interface would be worse than one that describes itself. ROADMAP.md
carries the full landed / outstanding split; the short version is that all six approved
design decisions are in, and the VRM texture loading, the local-first profiles and the
documentation are not.

- **Tokens.** `face.css` opens with a `:root` set — colour, type scale, spacing, radii —
  and every value in the file comes from it.
- **One drawer, four tabs** (Transcript, Settings, Health, Dev) replacing three
  hand-written drawers, each of which had its own header, close button and slide.
- **The API key lives in Settings**, not the status strip. A missing key lights an amber
  pill that opens Settings with the field focused — a guided empty state rather than a
  mystery.
- **Hush is transient**, inside the field, present only while there is something to stop.
- **The chrome sits in her light.** The day-cycle keyframes now carry the page's share of
  the hour, handed out through `onPalette` as `--room-tint`, `--room-ink` and
  `--room-line`. The window no longer floats in fixed grey above a room that moves.
- **The caption reaches distance size** (~34px from 25px), per the 10-foot-interface
  floor of about 28px at 1080p.
- **Status pills carry health dots** and the strip holds no controls.
- **"Listening Post" is now "In residence."**
- **The bust has a mouth, and eyes that are visible.** Both were buried inside the head —
  the mouth aperture closed to 1.6% of its height 0.076 behind the face surface, the iris
  reached 0.839 against a surface at 0.871. Lips are now deep forms whose front caps
  emerge and taper with the skull's curvature.
- **`setLightScale` joins the avatar interface.** A VRM is authored for roughly unit
  lighting; this room runs its key to 2.2 and clipped her to a white oval. Her material
  response is scaled instead of the room being dimmed, so the wall and the bust keep the
  light they were built for.

**Fixed:** a temporal-dead-zone error — the palette callback fires during
`createEnvironment`, before `avatar` exists — which stopped the scene building entirely.
Surfaced by `ready { faceBuilt: false }`.

## 0.9.2 — 2026-08-30

**She looks at you.** A camera button beside the microphone, wearing the same contract:
a person presses it, a marker stays visible the whole time it is on, pressing it again
ends it.

- While it is lit, her gaze follows movement — a **motion centroid over a 64×36 grid**,
  computed inside the renderer at ~8 Hz. Deliberately not a face detector: thirty lines
  a person can read, no vendored model, and it fails the way a person does — when
  nothing moves she keeps looking where she last saw you.
- **Nothing crosses the protocol.** No frame, no coordinate, no flag; the host cannot
  start it, stop it, or know it is happening. The only protocol change is `camera` in
  `hello`, so the face hides a button that could only fail.
- Two markers, two severities: the momentary red **bar** for a still, and a standing red
  **camera pill** beside the state pill for watching — the two facts a person needs at a
  glance share one corner.
- Verified live against the redirected webcam: button on → permission granted → pill up →
  head visibly turning with movement in the room; button off → device closed, pill gone.

## 0.9.1 — 2026-08-30

**The camera, against real hardware for the first time.** A webcam was redirected into the
VM, and everything 0.9.0 could only reason about became testable. Three things came out of
it, and none of them would have been found any other way.

- **The host now answers WebView2's permission requests.** There was no
  `PermissionRequested` handler at all, so the runtime decided — which made `"Camera":
  false` a suggestion rather than a boundary. The host now denies every permission except
  a camera request from her own origin with the setting on, and denies microphone,
  geolocation, notifications and the rest outright. Nothing is saved in the browser
  profile, so turning the camera off takes effect immediately.
- **`Glance` describes a captured frame without keeping it** — size, brightness and
  spread, logged on every `sight`. This is the silent microphone of Stage 4 all over
  again: a camera can open, report no error, and hand over a black rectangle, and from
  the outside that is indistinguishable from her being wrong about what she saw. A lens
  cap, a privacy shutter and an unlit room all look like success without it.
- **The capture waited two animation frames before grabbing.** Enough against a synthetic
  device, far too few against a real sensor that has auto-exposure to finish. Now 450 ms.
  Measured before and after on the same scene: brightness 0.15 → 0.18 — a real but modest
  improvement, and honestly the room is simply dark.
- The dev panel gains **Take a still**, which runs the whole camera path without needing a
  question that earns one. It is the only way to exercise the permission grant at all.

**Verified end to end:** device redirected → host granted the permission → `getUserMedia`
inside WebView2 → 768×432 frame with genuine detail (spread 0.126, well clear of the 0.02
blank threshold) → `sight` back to the host. The eyes are no longer half-verified.

## 0.9.0 — 2026-08-30

**Stage 9 — the attention gate, and eyes.**

MINOR: the gate is a new subsystem and it changes her behaviour in the most visible way
possible — she can now decline to answer.

### The gate

- **`AttentionGate`** decides whether something she overheard was addressed to her. Two
  layers, cheapest first: rules settle her name, the follow-up window and fragments for
  nothing; only ambiguous lines reach the small local model. **No paid model is ever used
  to decide whether to use a paid model.**
- **Fails open.** A companion who goes silent because a helper model died is broken; one
  who occasionally answers the television is annoying. The log says which happened.
- **Never silent.** A declined line is logged and sent to the face as `overheard` with
  the reason, and shown faintly in the transcript. "She ignored me" has to be answerable.
- **Typed input is never gated.** If you took the trouble to type it, you meant it.
- Measured over 18 labelled lines: 14 agreed, 1 ignored-you, 3 answered-noise, median
  1.2 s. `EarsTest -- gate` prints the table; `EarsTest` asserts the rules and the parser.
- Two findings that changed the design: the gate model must be the **same** model as the
  brain — a separate one is evicted and reloaded per utterance, 24 s against 0.7 s warm —
  and a **reasoning model is useless as a gate**, spending its whole budget deliberating
  and returning nothing. No portable switch turns that off.

### Eyes

- **`Situation`** replaces the loose `context` parameter on `IBrain.RespondAsync`, and
  now carries a still as well. It rides with the current question only, never the history.
- **The face owns the camera; the host owns the decision.** `look` asks for one frame,
  `sight` answers with it or with a reason. The device is released in the same breath and
  an unmissable marker shows while it is live.
- **Off by default — the only sense that is.** A microphone in a room is expected; a
  camera is not. Three cheap, auditable gates before it opens: the setting, the words, and
  whether the brain has eyes at all. None consults a model.
- `Sight.WantsEyes` is a word list rather than a classifier *on purpose*: a person must be
  able to read it and know exactly what makes her look.
- **Honestly half-verified.** The intent rules, the no-camera path, the refusal path and
  the marker are all tested. No frame has ever been captured here — this VM has no camera —
  so the picture reaching Claude is built and unproven.

### Also

- Self-test gains **Camera** and **Attention gate** checks; the latter fails loudly when
  the gate and brain models differ, because that misconfiguration is invisible and costs
  24 seconds an utterance.
- `Speech.WithoutThinking` for one-shot replies, alongside the streaming `ThinkFilter`.

**Not built:** the wake word (openWakeWord has no "Octavia", and the free layer already
matches her name for nothing), and presence detection, which needs the camera. Home
Assistant stays deferred by choice.

## 0.8.2 — 2026-08-30

**Stage 8's decision, and the contract that makes it possible.**

PATCH rather than MINOR on purpose: the stage is *decided*, not completed. Rendering waits
on a machine with a real GPU, and calling this a finished stage would misreport it.

- **`tools\attach-face.ps1 -Conformance`** drives a running host through a turn, a
  self-test and a forget, and reports which host-to-face messages arrived and whether each
  carried the fields `PROTOCOL.md` promises. Stage 8's premise — that photorealism is a
  renderer swap — was an assumption until something checked it.
- **It found a real gap on its first run.** A face attaching to a session already in
  progress was never told her current expression: `emotion` is only sent when her mood
  *changes*, and a mood can sit unchanged for many minutes. An external renderer would
  have shown the wrong face indefinitely. `hello` now carries `state`, `emotion` and
  `emotionWeight`, and the built-in face applies them on connect.
- **`PROTOCOL.md` gains "What a renderer must implement"** — what a face must handle, what
  it may ignore and what that costs, and the rates it has to survive. The checklist an
  Unreal face gets built against.
- The reconnection section now says what *is* replayed, rather than only what is not.

## 0.8.1 — 2026-08-30

**A dev panel: every performance she can give, on a button.**

Anything rare in the face — a mood, a viseme, the headphones — was awkward to look at,
because the only way to see one was to *cause* it. This is a fourth drawer that drives
`window.Face` directly, so a shape can be held still and judged.

- **State, mood, mouth, eyes, level, music, props and room**, each a row of buttons or a
  slider. `Say a line` runs a viseme sequence, because a single held shape says nothing
  about whether a mouth reads as talking.
- **`Hold the face`** stops host messages that would *move* her — `state`, `level`,
  `viseme`, `emotion`, `music` — from reaching the renderer, so a mood set by hand is not
  wiped by the next thing she says. Captions, the transcript and settings still arrive.
- **A Senses row that deliberately does leave the renderer**: listening, hush and the
  music sense are devices, and the face does not own one.
- **Offered on the `dev` profile, and whenever there is no host** — a face served on its
  own is being worked on by definition. `DevPanel` in config overrides both. The module
  is imported only when the panel is opened, so a published face never loads it.
- Three additions to the avatar-facing side of `window.Face` for things the face
  schedules for itself and could not otherwise be asked for: `blink()`, `look(x, y)` and
  `setProp('headphones', on)` — the last taking `null` to hand the prop back to the music.

## 0.8.0 — 2026-08-30

**Stage 7 — music: headphones on, dance.**

- **She hears what the machine plays.** WASAPI loopback taps the render endpoint, so she
  hears the output mix without a cable or a virtual device. This is the capability that
  stopped her being a browser page in the first place.
- **Beat detection with no model and no network.** A spectral-flux onset envelope,
  autocorrelated for a tempo, matched against a pulse train for the phase — the same
  arithmetic shape as the mouth in 0.7.0. `MusicAnalyzer` is device-free and pure, so it
  is tested against generated tracks at known tempi rather than by playing something and
  watching her.
- **Music, not speech.** Three things must agree: loud enough to be something, continuous
  enough not to be talking, and periodic at a steady rate. Speech fails the last two.
- **She keeps time while she talks.** Her own voice reaches the loopback like anything
  else, so the analysis is held while she speaks and the tempo already found keeps
  running — she stays in step with a track she is talking over, and cannot mistake
  herself for music.
- **The face responds**: headphones descend onto her head on sustained music, a nod on
  the beat, a sway across the bar, and the room's halo answers energy with a ring that
  leaves on each beat. `setHeadphones` joins the avatar interface; the bust and a VRM
  both implement it.
- **The brain is told there is music**, on the current question only — never in the
  system prompt, which would void the cache breakpoint, and never in the history, which
  would leave it claiming there is music an hour after it stopped. `IBrain.RespondAsync`
  grows an optional `context`.
- **`music` and `setMusic`** join the protocol as additive version-1 messages. No audio
  crosses it and none is kept: what survives analysis is a tempo and a loudness.
- **Settings**: a switch for whether she listens at all, off closing the device rather
  than ignoring it. The tempo appears in the status strip, because "she is not dancing"
  and "there is nothing to dance to" otherwise look identical.
- **Diagnostics**: a Music check naming the output device, and `Default output` in the
  report — "she never dances" is usually that line saying NONE.
- **`tools\serve-face.ps1`** serves `wwwroot` over loopback so the renderer can be
  developed in an ordinary browser with devtools, instead of a rebuild and a screenshot
  per change.
- Fixed on the way in: a `long.MinValue` sentinel that overflowed and stopped the tempo
  search ever running, and a beat clock that truncated fractional hops to zero so it
  never advanced while she spoke.

**Known limitation, and it is the machine's.** Remote Desktop's "Remote Audio" endpoint
normalises everything to full scale — crest factor 1.7, near square — at any volume. The
tempo cannot be found in audio with no dynamics left in it. `EarsTest -- music` measures
the crest factor and says so plainly rather than leaving it a mystery.

## 0.7.0 — 2026-08-30

**Stage 6 — a voice worth the face.**

- **`IVoice`**, alongside `IBrain` and `ISpeechRecognizer`. `VoiceBox` becomes
  `SapiVoice`; `NeuralVoice` joins it. `OctaviaSession` never learns which one it has,
  and the engine can be swapped under a running session.
- **Piper, out of process.** A long-lived child process: sentences on stdin, raw PCM on
  stdout, played through NAudio. Same reasoning as the local brain — a second ONNX
  runtime in this process would sit beside Whisper's CUDA-linked one. The 60 MB model
  loads once rather than once per sentence.
- The engine (22 MB) and the voice (~60 MB) are fetched on first use, into her data
  folder, with progress on her face. **This downloads an executable**, which is a
  different thing from downloading a model, so it happens only when the neural voice is
  asked for and the URL is in `PiperStore` where it can be read.
- **Lip sync is read out of the audio**, not from the engine. Piper reports no phoneme
  timings — and neither will most of its replacements. `VisemeReader` takes loudness for
  the jaw and the balance of three formant-ish bands for the lips. It is analysed at the
  moment each buffer reaches the sound card, so the mouth is in step with what is heard
  rather than with what has been generated.
  - Both its references adapt: loudness against a decaying peak, and the front/back axis
    against a running centre. Fixed thresholds tuned on one voice made every other voice
    mumble in a single shape.
  - `EarsTest -- mouth <file.wav>` prints the shape timeline, which is how the boundaries
    were set — a deliberately distributional choice, not a phonetic one.
- Settings → Speech chooses the engine; Settings → Voice lists that engine's voices and
  fetches one that has not been downloaded yet. `VoiceRate` maps to Piper's phoneme
  length, so speaking speed still works.
- She starts on the Windows voice and upgrades herself once the neural engine is ready,
  so a first run talks immediately instead of sitting mute through an 80 MB download.
- A small FFT in `Audio\Fft.cs`, written rather than taken from a package: thirty lines,
  no native dependency to collide with Whisper's, and Stage 7's beat detection wants the
  same thing.
- Protocol (still version 1): `setVoiceEngine` in; `hello` gained `voiceEngine`, and
  `voices[]` became `{value, label}` pairs because only the host knows how to tidy a
  Piper file name.
- Fixed: the end-of-utterance watchdog fired *during* synthesis, when the buffer was
  legitimately empty because nothing had been produced yet — she reached "idle" and then
  started talking. Nothing is over until it has begun.
- Fixed: a sound card is fed continuously and an empty buffer comes back as silence, so
  she sent a viseme twelve times a second forever, every one saying "mouth shut".
- Twelve voice checks added, and 15 face/expression checks moved to `SapiVoice`.

## 0.6.1 — 2026-08-30

**A settings menu, and the persistence bug it uncovered.**

- **Settings drawer** in the face: appearance (the bust or any `.vrm` in her avatars
  folder), voice, and the room's lighting hour. Changes apply instantly and are saved.
  Voice moved here from the console row, which now shows it as a label.
- The host lists what is actually in the avatars folder, so choosing a character is a
  dropdown rather than a filename typed into a config file. A name that is not there is
  refused rather than saved.
- `RoomHour` in config.json pins the room's lighting; negative follows the clock.
- Protocol (still version 1): `setAvatar` and `setRoomHour` in; `hello` gained
  `avatars[]`, `avatarFile` and `roomHour`.
- **Fixed: settings did not persist.** v0.4.1 stopped `Save()` flattening the profile
  overlay into the file by carrying back a *hand-kept list* of runtime-changeable
  properties — which was wrong the moment a new setting existed. `AvatarFile` and
  `RoomHour` reached the host, changed the face, logged, and were silently dropped on
  save. `Save()` now writes back every key that differs from the settings as they stood
  at load, which is the same guarantee without a list to keep in step.
- Fixed: a face that cannot reach the avatar origin retried on every `hello`, refetching
  megabytes and logging an error each time. A URL that failed is not retried.
- Fixed: switching avatars only ever loaded; it never switched back to the bust.
- Five config checks added, covering a new setting persisting, two saves in a row, and
  the overlay still not leaking.

## 0.6.0 — 2026-08-30

**Stage 5 — the real face: VRM avatar and a room to stand in.**

- **three.js r180, as ES modules.** The vendored 2021 UMD build could not host
  `@pixiv/three-vrm` (r158+), and writing the new scene against the old API would have
  meant writing it twice. `three`, `GLTFLoader`, `BufferGeometryUtils` and `three-vrm`
  are vendored under `wwwroot\lib` with their bare specifiers rewritten, so no import map
  is needed and the CSP stays `script-src 'self'`.
- **One avatar interface** — `setViseme`, `setExpression`, `setGaze`, `setBlink`,
  `setPose`, `update`. The plaster bust implements it; so does a VRM. The face owns blink
  schedules, saccades, head carriage and mood; the avatar owns how a jaw actually moves.
- **VRM characters.** Drop a `.vrm` in `%APPDATA%\Octavia\avatars` and name it in
  `AvatarFile`. The host maps that folder to a read-only `https://octavia.avatar` origin
  and offers the URL in `hello`; the face loads it once and falls back to the bust —
  loudly, into the log — if anything goes wrong. Arms are posed out of the format's
  T-pose on load, since VRM supplies a rest position rather than an idle.
- **The expression vocabulary is VRM 1.0's**, deliberately: `happy / angry / sad /
  relaxed / surprised / neutral`, and visemes `aa / ih / ou / ee / oh`. Protocol to
  character is an identity mapping with nothing in between to get wrong.
- **Visemes carry a shape.** SAPI's 21 identifiers were collapsed to one openness number;
  they now also map to a mouth shape. "aa" and "ou" are the same jaw drop with different
  mouths, and that difference is most of what makes speech look like speech.
- **`emotion` message.** Her expression is read from the text of each sentence as she
  speaks it — locally, free, no model call, per the standing rule that reflex-speed things
  stay local. The message exists so a model can override it later.
- **A room, not a backdrop.** The flat wall became a shader environment: a full-day
  lighting cycle (the wall's temperature and the key, rim and ambient lights move
  together), two drifting depth slabs for parallax, a vignette, grain, and a halo behind
  her that answers the microphone. `Face.setHour(21)` pins the clock to look at it.
- Self-test gained an **Avatar** check; "she looks wrong" becomes a filename.
- Fixed: the backdrop's vignette used `smoothstep` with its edges reversed, which is
  undefined in GLSL and rendered as no vignette at all.
- 15 face-and-expression checks added to `tools\EarsTest`, covering the viseme map's
  coverage and the mood reader's vocabulary.

## 0.5.0 — 2026-08-30

**Stage 4 — diagnostics: make her debuggable in someone else's hands.**

- **Structured logging.** `octavia.log` gains levels (`debug`/`info`/`warn`/`error`),
  rolls at 1 MB keeping three predecessors, and remembers its recent lines in memory so
  the face can show them without touching the disk. `LogLevel` in config.json;
  `OCTAVIA_LOG` writes it elsewhere.
- **Real crash handling.** The UI thread's unhandled exceptions were logged and swallowed;
  they are now logged with a stack trace *and shown on her face*. Background-thread and
  unobserved-task exceptions are caught too.
- **Self-test**, in-app and on demand: settings, transports, renderer, microphone signal,
  speech model, voice and brain. Every failing check carries the sentence that says what
  to do about it. Deliberately free — the local brain is pinged, Claude is never called.
- **"Save diagnostics"** — a file dialog and one zip: `README.txt`, `report.txt`,
  `config.json` with anything key-shaped removed, and `logs/`. Reachable from the face and
  from the tray, because the moment you most need it is when the face is what broke.
- **Privacy, stated up front.** The log contains transcripts of things said in the room.
  README.txt says so, names the file to read first, and confirms the API key is not in the
  bundle — it stays DPAPI-sealed outside it.
- **`--diagnostics <path>`** writes a bundle with no window and no session, so a machine
  where she will not start at all can still produce one. It runs before the single-instance
  check on purpose.
- A **Health panel** in the face: the checks, this machine's facts, and the recent log.
- Fixed: the save dialog was constructed on a socket thread, so it threw into a discarded
  task and did nothing whatsoever. Every fire-and-forget task now logs its own failure
  instead of waiting for the garbage collector to notice.
- Fixed: the headless bundle blocked the dispatcher thread on a task whose continuations
  were posted back to it, and hung forever.
- Fixed: the config redactor matched substrings, so it blanked `Hotkey` and `MaxTokens` —
  two of the most useful lines in a fault report. It now matches whole words of the
  setting's name and only ever redacts a *string*, since a secret is never a number.
- Protocol (still version 1): `selfTest`, `saveDiagnostics`, `openDataFolder` in;
  `diagnostics`, `diagnosticsSaved` out. A face may ask for a bundle but never names the
  path — the host owns the dialog.
- `tools\attach-face.ps1 -Send '<json>'` sends arbitrary protocol messages, and prints
  self-test results.
- 27 diagnostics checks added to `tools\EarsTest`, covering levels, rotation, the check
  set, and every guarantee the bundle makes about its own contents.

## 0.4.1 — 2026-08-30

**Profiles you can pin to a launcher.**

- `--profile <name>` (also `--profile=<name>` and `-p`) on the command line, outranking
  `OCTAVIA_PROFILE` and the `Profile` key in turn. A desktop shortcut can pass an
  argument but cannot set an environment variable, so without this a launcher had no
  way to say which rig it wanted.
- The desktop shortcut now passes `--profile dev`, so it always starts on the local
  model regardless of what the config file happens to say.
- The startup log records where the profile came from — `profile 'dev' (command line)`
  — and the tray tooltip reads `Octavia — dev (local)`.
- Fixed: saving a setting while a profile was applied wrote the *merged* values back,
  flattening the overlay into the base settings permanently. Changing her voice on the
  dev profile therefore rewrote the file's base brain to `local`, and every later run
  inherited it. Runtime changes now go back to the un-overlaid original.
- Launching a second instance while she is running ignores `--profile`; it now says so
  in the log instead of silently surfacing an Octavia on the other profile.
- `OCTAVIA_CONFIG` points her at a different settings file, so the harness can exercise
  loading and saving without touching the real one.
- Twelve config checks added to `tools\EarsTest` covering the precedence order and the
  flattening regression.

## 0.4.0 — 2026-08-30

**Stage 3 — cut the cord: the face protocol.**

- `PROTOCOL.md` — the host/face contract written down, with a `protocol` version carried
  in `hello`. Faces must ignore unknown types and fields; removing or repurposing one is
  a version bump.
- `WebSocketFaceServer` — a loopback listener (raw `TcpListener` + `WebSocket.CreateFromStream`,
  so no urlacl reservation and no elevation). Binds `127.0.0.1` only and requires a
  per-run token, compared in fixed time. Bad or missing token is refused at the handshake
  with 401 and never becomes a WebSocket.
- `FaceHub` — fans one message out to every attached face and merges what comes back.
  The session no longer knows how many renderers are listening or which transport each
  chose.
- The built-in page now prefers the socket too, so it is no longer a special case; it
  falls back to postMessage only if the port could not bind. Confirmed that a page on a
  virtual `https` origin *can* reach `ws://127.0.0.1` — loopback counts as potentially
  trustworthy, so mixed-content blocking does not apply.
- `tools\attach-face.ps1` — attach to a running Octavia as an external face and drive her,
  proving the protocol rather than the WebView2 page is the interface.
- Eight protocol checks added to `tools\EarsTest`, including both token-refusal cases and
  fan-out to two simultaneous faces.
- Fixed: the server abandoned the socket on a close frame instead of completing the
  handshake, so a face that disconnected politely saw an EOF error on its way out.
- Config: `FacePort` (default 8848; 0 picks any free port).

## 0.3.0 — 2026-08-29

**Stage 2 — a local brain, and dev profiles.**

- `IBrain` interface; the old `Brain` becomes `ClaudeBrain`, joined by `LocalBrain`.
- `LocalBrain` streams from any OpenAI-compatible server (Ollama, LM Studio,
  `llama-server`) over SSE, so swapping models is a config edit, not a rebuild.
  Kept out-of-process on purpose — a second CUDA-linked native runtime in this
  process would collide with Whisper, and later with Audio2Face.
- Shared `Conversation` and `Speech` helpers: sentence draining, a markdown
  flattener, and a streaming `<think>` filter so a reasoning model's scratchpad is
  never spoken aloud.
- Named config **profiles** merged over the base settings in memory; `OCTAVIA_PROFILE`
  overrides the file. `dev` = local brain + `small.en`; `live` = Claude +
  `large-v3-turbo`.
- Ollama installed and benchmarked on the dev VM; `llama3.2:3b` chosen as the dev
  default on wall-clock and persona adherence, not tokens/sec.
- Silence watchdog: a microphone that opens but delivers digital silence now says so
  on her face after 10 seconds instead of failing invisibly.
- `tools/EarsTest` gained 16 brain checks, a live local-brain probe, and a
  `-- mic` device diagnostic.
- Fixed: the `<think>` filter held a fixed lookahead margin, so replies shorter than
  the tag were never counted as spoken and `LocalBrain` threw "returned nothing".

## 0.2.0 — 2026-08-29

**Stage 1 — ears: VAD + Whisper.**

- `WhisperRecognizer` behind the existing `ISpeechRecognizer`: microphone →
  Silero VAD → Whisper, entirely local.
- Silero VAD (vendored ONNX) gates every utterance with pre-roll, hangover and a
  minimum voiced duration, so Whisper never sees silence and cannot hallucinate
  text out of it. Bracketed non-speech tags are filtered as a second line.
- Whisper models download once to `%APPDATA%\Octavia\models`, with progress on her
  face. CPU and CUDA runtimes both referenced; CUDA is picked up automatically.
- The Windows desktop recognizer stays as an automatic fallback.
- `tools/EarsTest` added: synthesizes speech, runs the whole pipeline headlessly,
  and asserts that silence transcribes to nothing.

## 0.1.0 — 2026-08-29

**Initial build.** Grew out of a single-file HTML prototype (`talking-avatar.html`).

- WPF / .NET 10 host with the face in a WebView2, served from a virtual
  `https://octavia.face` origin so the page is a secure context.
- Three.js plaster bust ported out of the prototype into `wwwroot`, driven entirely
  by host messages behind `IFaceTransport`.
- Claude via the Anthropic SDK, streamed and cut at sentence boundaries so she starts
  speaking before the reply is finished.
- API key sealed with DPAPI to the current Windows account — it never reaches the page.
- SAPI speech synthesis with real viseme events driving the jaw.
- Tray icon, configurable global hotkey, single-instance with window surfacing.
```
