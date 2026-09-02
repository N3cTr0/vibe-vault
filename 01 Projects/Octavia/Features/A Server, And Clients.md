---
project: Octavia
tags: [octavia, feature]
---

# A Server, And Clients

*Stage 15 item 1, v0.26.0.* She stops being a window that contains a being, and becomes a
being that windows connect to.

Built from [[Stage 15 - Server And Clients]], which was written first — in her own repo
rather than from the [[Octavia Android]] side, because for once the consumer that needed it
was the desktop.

## The ask

> Since Octavia runs as the host and the android app is technically a client, what would it
> take to convert this project into a server/client setup, where the AI and all the actual
> work gets moved to a different server `.exe`… and then have a client version on windows
> like how she is now just connect onto that instance, the same with the android app.

## Most of it was already built, twice, under other names

This is the interesting part, and it is entirely a consequence of [[Architecture|the rule]]
having been kept. Three findings, each an accident of doing an earlier stage properly:

| | |
|---|---|
| **The session did not know WPF existed** | Three references to a UI framework in 9,361 lines, and only one inside `OctaviaSession` |
| **`FaceHub(page, sockets)` already took a nullable page** | A server is that argument being null. Not a refactor — a call site |
| **`RoomChecks` had been running her headless for a version** | A real session against a recording transport, no window, fifty assertions |

> **A test harness had accidentally been a server for a month.** That is the strongest
> evidence a seam is in the right place that this project has produced: nobody set out to
> prove the session could run without a window, and it had been proving it on every run.

## What had to be answered

Three things could not simply move, and each is more interesting than the move was.

### A file dialog cannot survive a headless host

`saveDiagnostics` raised a `SaveFileDialog`, which needs a dispatcher and somebody looking at
it. So **the one control that exists for when she is broken would have been broken by moving
her** — and silently, on the machine where it mattered.

It writes into `data\diagnostics\` and answers with the path over the socket. That is better
than what it replaced rather than a concession: the old dialog put the bundle only where the
person at *that* machine chose, which is no use when the machine that needs diagnosing is in
another room.

It stays host-only. There is no longer a technical reason for it to be — but the bundle
carries her log, her config and a system report, and widening the authority table is a
decision of its own rather than a side effect of moving a file dialog.

### The page had a second transport with nothing left at the end of it

`postMessage` existed only for the WebView2 page hosted inside her own process. Left in, it
would have been actively harmful in two ways: `send` would have reported success into a void,
and — worse — the *"lost the connection"* notice was **suppressed when a WebView2 host was
present**, which after the split is exactly the face that needs it.

### Which meant the page had to learn to reconnect

While she *was* the window, a dead socket meant a dead application and there was nothing to
reconnect to. A server restarts on its own: an upgrade, a crash, a machine waking up.

There is a persistent bar now — *lost her — reconnecting* — because a notice fades after
seven seconds and a face that has quietly stopped being connected then looks exactly like one
that has not. Backoff runs 500 ms to 15 s: quick enough that a server restart is barely
visible, slow enough that a face left running against a machine that is off for the night is
not talking to itself all night.

> **`ready` is re-announced on every open, not only the first**, and this is the rule that
> would have been easiest to miss. A reconnected face is a genuinely *new* face to the host:
> new `FaceId`, no room, no senses, nothing remembered. A renderer that announced once would
> find itself silently back in the host room with no camera — working, apparently, and wrong.

## The checks changed more than the code did

Both suites that drive the real page had been *pretending to be the host over `postMessage`*.
With that channel gone they speak the protocol instead, through a real `WebSocketFaceServer`.

That is a fairer test than the one it replaces, and it earned its keep immediately: **it is
how the "reconnected faces must re-announce" rule was found**, because the first thing that
broke was a page with no socket never announcing at all.

`EarsTest -- split` is new, and checks the boundary as *text* against the source: the client
never constructs a session or a brain, the core never reaches for a file dialog or
`Application.Current`, the page has one transport and reconnects, one version covers all
three assemblies. A compiler cannot express any of that, because all three assemblies
legitimately see each other's internals — so the rule is enforced where it is actually
written down.

Broken on purpose first: four went red, sixteen came back green. 288 assertions pass overall.

## Three assemblies, one program

`InternalsVisibleTo` rather than a public surface, and deliberately. A public API here would
advertise something nothing outside this repository will ever call, and would force `public`
onto forty types the design keeps closed. **The server and the client are not consumers of a
library; they are the same program split for deployment.**

The client authenticates with the **remote key**, not the per-run token — the token is minted
fresh on every start, so a client holding one would need re-pairing every time she restarted.
That is the same reason a phone uses the key, arriving from the other direction, and
`Authorised` has always accepted it from loopback.

`Hotkey` and `StartMinimised` moved to `client.json`, carried over from her settings on first
run so nobody loses a key combination they chose. They always described *this* Windows session
and *this* window; neither means anything to a server.

## What this deliberately did not do

**The host room still means "the room the server is standing in".** While the server runs on
the machine with the microphone and the speakers, that is *correct* rather than merely
convenient — it really is standing in one. The definition only breaks when the box moves to a
cupboard, and paying for it in the same change set would have meant neither half could be
tested alone.

> **The practical consequence, today:** run the client on the server's machine, or use the
> neural voice. Her voice plays through the *server's* sound card for the host room, and a
> Windows voice cannot be streamed to a client at all. See [[The Voice]].

The answer when it comes is already written: the client becomes a privileged face that lends
the server its devices — which is [[Lending A Renderer The Device's Senses]] moved up one
level, *renderer* swapped for *server*. That it falls out of a seam which already exists is
the best evidence there is that the shape is right.

## The cost, stated before it was paid

- **Security stopped being optional.** `RemoteAccess` was opt-in and the desk worked without
  it. The socket is now the only way in, so the bearer key over plain HTTP is load-bearing —
  on a machine whose firewall is off entirely. See [[Conventions & Security Model]].
- **One process, one log, one breakpoint** was what made her pleasant to debug. *"She didn't
  answer"* now has a second place to hide.

## Links

- [[Stage 15 - Server And Clients]] — the specification, written before the code
- [[Architecture]] — the rule that made this a move rather than a rewrite
- [[One Being, Many Rooms]] — rooms, and what the host room now has to mean
- [[Lending A Renderer The Device's Senses]] — the seam item 3 generalises
- [[Face Protocol]] — unchanged by this stage, which is the point

## She runs as a service *(v0.30.0 — Stage 15 item 4, closed)*

*The client half landed in v0.28.2. This is the other one, and it stayed on the list because the owner corrected the reasoning that would have removed it: **"it may not always be the case"** that the server and the client share a box.*

```
Octavia.Server.exe --install --profile home
```

Auto-start, so she is there after a reboot with nothing double-clicked. **Start Octavia** and **Stop Octavia** are desktop shortcuts. `--uninstall` removes her; the console is untouched and still the right thing to run when you want to *watch* her start.

### No administrator, which was the point

A service is normally an administrator's object, so the obvious build makes every start and stop raise a UAC prompt — between somebody and their own companion, forever.

`--install` splices **one** entry into the service's own security descriptor: `RPWPDTLOCRRC` for the installing account — start, stop, pause, query, and nothing else. Deliberately **not** `WD` or `WO`; the right to hand *other people* control of her stays with administrators.

The descriptor is **read and spliced, never rewritten**. A hand-written SDDL that happens to omit an entry Windows put there is how a service becomes unmanageable by the system that installed it.

### The clean shutdown, at last

[[Lessons Learned|Three mechanisms were measured failing]] to stop the console server from outside: `CloseMainWindow` skipped the unwind entirely, Ctrl+C was delivered and ignored, and Ctrl+Break could not be survived by whoever raised it.

`--stop` produced **"Octavia server stopped"** on the first attempt. Stopping a service is something Windows is *designed* to do; closing somebody's console window is not. That is the argument for this item restated as a measurement — and it is why the client still refuses to stop her itself.

### Two things found by building it

**The registered path was stored unquoted.** `sc create` reported success, the service ran, and the `ImagePath` in the registry was missing the quotes around the exe — because the command was built as one string and two layers of parsing disagreed. Invisible under `C:\Projects`; the classic unquoted-service-path failure under `C:\Program Files`, where Windows resolves `C:\Program.exe` and the service never starts. Found by reading the registry back rather than trusting the call. `ArgumentList` fixed it.

**A service runs as LocalSystem, so the hosted brain has no key.** `apikey.dat` is DPAPI-sealed to a *user account*. The local brain — the default, and the profile she actually runs on — is unaffected. `--install` prints this at the moment somebody is definitely looking, with both fixes: a machine-wide `ANTHROPIC_API_KEY`, or logging the service on as the user in `services.msc`.

### Small things that follow from session 0

The single-instance mutex is `Local\`, so it cannot see across sessions — a service and a console would both pass it. The real guard is the port, and a console that cannot bind now says whether the *service* is what holds it, which is a cause it could not otherwise have seen.

`LocalServer` asks `--start` before spawning a console of its own, because a service outlives the window that wanted it and comes back after a reboot.

**Not done:** the same start and stop in the client's tray. The shortcuts answer what was asked; the tray is a nicety that can follow.
