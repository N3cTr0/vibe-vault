---
project: Octavia
tags: [octavia, spec, stage-15]
---

# Stage 15 — A server, and clients

> **Item 1 built, 09/02/2026, v0.26.0.** Added by the repo side above the original text,
> which is untouched below. Acceptance 1, 2, 3, 4, 5, 7, 8, 9 and 10 are all confirmed —
> 288 assertions pass, the renderer conformance check passes against the server, and the
> client was watched losing its server and finding it again. **Acceptance 6 is owed**: the
> protocol did not change and the LAN listener is up, but no handset was in the room to
> prove it. Items 2–5 are open and recorded in `ROADMAP.md`. See
> [[A Server, And Clients]] and [[Changelog]] 0.26.0.

> **A specification written before the code**, 09/02/2026, in her own repo rather than
> from the [[Octavia Android]] side. It follows [[Stage 14 - Lending A Renderer The Device's Senses]],
> which closed item 10. The owner asked for the structure first — *"we can figure it all
> out once we have a server / client setup"* — so this deliberately splits the process and
> defers the questions that only matter once it is split.

## The ask, in the owner's words

> Since Octavia runs as the host and the android app is technically a client, what would it
> take to convert this project into a server/client setup, where the AI and all the actual
> work gets moved to a different server `.exe` that manages all the AI stuff etc, and then
> have a client version on windows like how she is now just connect onto that instance, the
> same with the android app. I think doing it this way would allow the coding to be more
> optimized later down the road and adding an MCP server etc.

## Most of this is already built, twice, under other names

That is the finding, and it is what makes the stage small enough to attempt in one change
set. Three pieces of evidence, all of them accidents of doing earlier stages properly.

**The session does not know WPF exists.** Across 9,361 lines there are three references to
a UI framework outside the window itself:

| Where | What | Answer |
|---|---|---|
| `Face/WebViewFaceTransport.cs` | `Dispatcher` | It *is* the UI transport. Goes to the client. |
| `Brain/Situation.cs:85-91` | `BitmapFrame` to grey a camera still | Stays; see *What is not portable yet*. |
| `OctaviaSession.cs:937` | `Application.Current.Dispatcher` + `SaveFileDialog` | **Must be answered.** See below. |

`OctaviaSession.cs` is 1,701 lines and only the third one touches it.

**The transport already composes as a headless host.** `FaceHub(WebViewFaceTransport? page,
WebSocketFaceServer? sockets)` — both nullable. A server is `new FaceHub(null, sockets)`.
That is an argument, not a refactor.

**She has already run headless, and it is green.** `tools/EarsTest/RoomChecks.cs` builds a
real `OctaviaSession` against a `RecordingFace : IFaceTransport` with no window, no WebView2
and no dispatcher, and drives fifty assertions through it. **That harness is a server minus
a listener.** The thesis was proven months before it was proposed.

Add [[Face Protocol]] gaining rooms, identity and per-face addressing in item 9, and the
socket already serving `wwwroot` over HTTP with a credential since v0.20.0, and the wire is
finished. What is missing is packaging and one decision.

## The shape

```
src/
  Octavia.Core/        class library  — session, brain, rooms, senses, voice, sockets, wwwroot
  Octavia.Server/      console exe    — headless: owns data/, config, log, the listener
  Octavia.App/         WinExe (WPF)   — the client: window, WebView2, tray, hotkey
```

`Octavia.App` keeps its name, its assembly name, its icon and its manifest, because it is
still the thing a person double-clicks. It simply stops containing her.

**The client authenticates with the remote key, not the per-run token.** `Authorised`
already accepts either from loopback, and the key survives restarts where the token does
not — so a client that starts before the server, or outlives one, does not need re-pairing.
This is why `RemoteKey` being deliberately readable turns out to matter more than it did.

## The one decision this stage does *not* make

**What is the host room when there is no host?**

`RoomId.Host` means "the room the process is running in". Move the process to a headless box
and that definition evaporates: `listen`, `setMicrophone`, `setOutput`, `setMusic` and the
loopback are commands about a machine nobody is standing in.

Two answers:

- **(a) Abolish it.** Every face is a room; the local-device features die with the desk.
- **(b) The client lends the server its devices.** The Windows client becomes a privileged
  face declaring `senses: ["mic", "camera", "speakers", "loopback"]`, and the server routes
  device-shaped work to it.

**(b) is `window.OctaviaEmbedder` moved up one level** — item 10's sentence with *renderer*
swapped for *server* and *embedder* swapped for *the client on the machine with the
hardware*. That it falls out of an existing seam is the strongest evidence the architecture
is right, and it is also a whole stage of work.

> **Neither is built here, and that is deliberate.** For as long as the server runs on the
> machine with the devices — which is the only deployment that exists today — the current
> meaning is *correct*, not merely convenient: the host room is the room the server is
> standing in, and it really is standing in one. The definition only breaks when the box
> moves to a cupboard, and that is when it should be paid for. Splitting the process and
> redefining the host room in one change set would mean neither could be tested alone.

Recorded as **item 3** below, open.

## What has to change, and why each one is not optional

### 1. `saveDiagnostics` cannot open a file dialog

A server has no desktop, and `SaveFileDialog` needs both a dispatcher and somebody looking
at it. The bundle is written into `data/diagnostics/` under a timestamped name and the path
is announced with the existing `diagnosticsSaved` message.

This is **better than what it replaces**, not a concession. The dialog put the bundle only
where the person at *that* machine chose, which is no use when the machine that needs
diagnosing is in another room; the path now goes back over the socket to whoever asked.

`saveDiagnostics` **stays host-only**. There is no longer a technical reason for it to be —
but the bundle carries her log, her config and a system report, and widening the authority
table is a decision of its own rather than a side effect of moving a file dialog.

`openDataFolder` keeps `explorer.exe` but is a no-op with a logged line when there is no
shell — a server in a cupboard has nowhere to open a window.

### 2. The client stops being exempt from her voice

Today:

> *Only socket faces can receive audio. The built-in page shares this machine's speakers, so
> streaming to it would be her talking over herself in the same room.*

After the split there are no speakers on the server, so **the Windows client becomes an
audio sink exactly like the phone**. It opts in with `subscribe`/`want` like anything else,
and `wwwroot` already plays her voice — the phone has been doing it since item 9.

Consequences worth stating rather than discovering:

- `SapiVoice` cannot be streamed and therefore cannot be heard through a client. She already
  says so out loud in a `notice`; that notice now fires for the desk too. **The neural voice
  stops being an upgrade and becomes the way she talks.**
- The viseme tap in `MouthTap` now sits a network hop from the ear that hears it. Sub-
  millisecond on loopback and on a LAN, and the visemes and the PCM still leave the same
  buffer at the same instant — but the standing constraint *"anything reflex-speed is
  local"* now means *local to the renderer*, not *in-process*. Re-stated, not broken.

### 3. `bridge.js` has to survive the server restarting

One `new WebSocket(address)` at line 136 and no reconnect. Today that is correct: the socket
dies when the process dies and the process *is* the app. After the split the server restarts
independently and **every client goes dark permanently with nothing on screen to say so** —
which is exactly the failure class [[Lessons Learned]] keeps recording.

Reconnect with backoff, and the page must say which state it is in. A face that is trying is
a different thing from a face that has given up.

### 4. Every `internal` becomes a cross-assembly concern

The whole codebase is `internal` with one `InternalsVisibleTo`. Splitting into three
assemblies forces a choice per type. The rule taken here: **`public` only where the client or
the server genuinely reaches**, and `InternalsVisibleTo` for `EarsTest` on the new core so the
checks keep testing internals rather than the split forcing a wider surface than the design
wants.

## What is not portable yet, and is worth saying out loud

`Octavia.Core` still targets `net10.0-windows` with WPF on, for exactly one reason:
`Sight.Inspect` decodes a camera still with `System.Windows.Media.Imaging`. `BitmapFrame`
works headless — it is WIC underneath and needs no dispatcher — so this costs nothing today.

Everything *else* in the core is already portable in principle. Whisper.net and ONNX Runtime
are cross-platform; NAudio and `System.Speech` are not, but they are already behind
`IAudioSource`, `ISpeechRecognizer` and `IVoice`.

> **So the Linux server is one image decoder and one `Octavia.Windows` project away**, and
> that is the prize worth naming: the always-on box in a cupboard wants to be Linux. Not
> built here. Recorded as item 2.

## Acceptance

Each is checkable from outside, and the ones marked ✽ are asserted in `EarsTest`.

1. `Octavia.Server.exe` starts with no window, no WebView2 and no WPF `Application`, and
   logs a bound port.
2. A browser on the LAN reaches her page from the server alone, holds a conversation, and
   hears her voice. ✽ *(the server half)*
3. `Octavia.App.exe` starts with **no session in it** — grep proves it constructs no
   `OctaviaSession`, no brain, no recogniser — connects to the server, and is
   indistinguishable from v0.25.1 to look at.
4. The desktop client hears her voice through the socket, and her mouth still matches it.
5. Killing the server leaves every client saying so on screen; restarting it reconnects them
   without anybody touching a browser. ✽ *(the page's state machine)*
6. The Android client connects to the server unchanged — **no client edit, no protocol
   change** — and rooms still work: the phone is in `phone`, the desktop client in `host`.
7. `saveDiagnostics` from any room writes a bundle and reports its path; the zip is byte-
   identical in content to one written before the split. ✽
8. Every existing `EarsTest` check passes against the new assemblies, untouched. ✽
9. The host room's meaning is *unchanged* — `listen` and the device settings still work from
   the desktop client and are still refused from the phone. ✽
10. Two clients on one server see one being: same brain, same voice, same key, different
    rooms and different moods.

## What this stage does not do

- Does not answer the host-room question (item 3).
- Does not make the core portable (item 2).
- Does not add an installer, a service registration, or auto-start for the server.
- Does not change the protocol. **If a client edit turns out to be needed, that is a
  finding, not a step** — the whole claim of Stage 3 is that this is possible without one.

## Open items

| # | | State |
|---|---|---|
| 1 | The split itself — three projects, a headless server, a thin client | *this stage* |
| 2 | A portable core: decode a still without WPF, `Octavia.Windows` for NAudio and SAPI | open |
| 3 | What the host room means when the server has no devices — the client lends its senses | open |
| 4 | The server as a Windows Service, with the client starting it on demand | open |
| 5 | Diagnostics bundles downloadable over HTTP rather than only written to disk | open |

## The cost, stated before it is paid

- **Security stops being optional.** `RemoteAccess` is opt-in today and the desk works
  without it. After this, the socket is the only way in, so the bearer key over plain HTTP
  becomes load-bearing — on a machine whose firewall is currently off entirely. See
  [[Conventions & Security Model]].
- **One process, one log, one breakpoint** was the thing that made her pleasant to debug.
  Two processes means correlating two logs across a socket, and *"she didn't answer"* gains
  a second place to hide.

Neither is a reason not to. Both are reasons to have written this down first.

## Links

- [[Stage 14 - Two Rooms]] — item 9, which made a face addressable and gave rooms meaning
- [[Stage 14 - Lending A Renderer The Device's Senses]] — item 10, the seam item 3 generalises
- [[Architecture]] — the `IFaceTransport` seam this rides on
- [[Face Protocol]] — unchanged by this stage, which is the point
- [[Octavia Android]] — the client that must not need editing
