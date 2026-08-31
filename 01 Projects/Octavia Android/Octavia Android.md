---
project: Octavia Android
tags: [octavia, octavia-android, moc]
---

# Octavia Android — Project Hub

> A second face for [[Octavia]], over the protocol her built-in one already speaks. She stays on the PC at home; this is an Android window onto her — a WebSocket, a renderer and (eventually) a microphone, reached over WireGuard. **This note is the reference point for everything on the phone side.**

- **Repo:** `C:\Projects\Octavia-Android` — git since 08/31/2026, private remote [`N3cTr0/Octavia-Android`](https://github.com/N3cTr0/Octavia-Android)
- **Current version:** 0.3.0 — **Stage 1 works**: it connects to her and you can talk to her from the phone. AGP 9.0.1, Kotlin 2.3.20, `minSdk 28`, `targetSdk 36`, package `com.n3ctr0.octavia`
- **Started:** 08/31/2026, out of [[Octavia]]'s Stage 13
- **Parent project:** [[Octavia]] — her `PROTOCOL.md` is the contract and lives in *her* repo, deliberately not copied here
- **Written against:** protocol version 1, as of her v0.19.3

## The one idea to keep

**This is a renderer, not a second Octavia.** No API key, no model, no conversation state, no decision about what she says. Everything it knows arrived as a JSON message. That is not a rule invented for the phone — it is the same separation that lets a plaster bust become a VRM without `OctaviaSession` changing shape. See [[Architecture]] in her folder.

## Why this is a separate repo

Her Stage 13 says so explicitly: the app needs an Android SDK and a device to test on, and her repo is neither. The more useful reason is that keeping it separate keeps the protocol honest — anything the phone needs has to be a *protocol* change rather than a quiet reach into her internals, which is exactly the pressure that keeps the socket a real interface.

## Where it actually got to

**Stage 1 is working on the real J7 Pro (08/31/2026).** She answered a typed question from the handset, spoke the reply on the PC, and both turns came back to the phone. Her log read `face connected over socket (2 attached)` and `face skipping: viseme, level` — a **second face beside her desktop one**, which is exactly what Stage 3 built the protocol for. See `Screenshots\`.

**The development loop needs nothing exposed:** `adb reverse tcp:8848 tcp:8848` points the handset's own loopback at the host's, over USB. Her socket stays bound to `127.0.0.1`, `RemoteAccess` stays off, no Windows firewall rule and no elevation — and because the connection arrives from loopback, the per-run token works. Wireguard and the remote key replace only the address.

Three decisions from Stage 1 worth keeping:

- **`org.json`, not a generated parser.** The protocol requires a face to ignore types and fields it does not recognise; defensive `opt*` reads say that directly, where a schema-bound parser turns her adding a field into this app crashing. **OkHttp is the only third-party dependency** — Android has no WebSocket client, and hand-rolling RFC 6455 to avoid one library is a poor trade.
- **The typed line is not echoed locally.** She sends `turn who:"you"` back *before* she starts thinking, so a local line would double it — and waiting for hers means the transcript also shows what was said at the desktop.
- **`say` carries `text`, not `value`.** Checked against `OctaviaSession` rather than the table, because the wrong field is accepted and silently ignored.

### Older note, from before anything was built

Nothing is built. The repo, the design and the version history exist; there is no Kotlin, no Gradle project, and no Android toolchain on this machine.

That is not a stall — it is the order her roadmap asks for. Stage 13's steps 1–4 all shipped in her **v0.14.0**: the socket can bind the LAN (`RemoteAccess`), `remote.key` survives a restart, `subscribe` lets a phone decline sixty visemes a second, and the network decision is the UDM SE's own WireGuard.

## The two prerequisites found on 08/31/2026

Both live in **her** repo, and neither was visible until the renderer question was answered. Full write-up in `ROADMAP.md` stage 0 in this repo.

### The face page is not served over HTTP

`wwwroot` reaches the built-in face through `CoreWebView2.SetVirtualHostNameToFolderMapping` — a **WebView2 feature, not a server**. `WebSocketFaceServer` is a raw `TcpListener` that only ever speaks the WebSocket upgrade, chosen that way in Stage 3 to avoid a urlacl and elevation. So step 1 opened the *socket* to the LAN and left the *page* where it was, and an Android WebView has no equivalent of a virtual host mapping.

- **(A) Serve `wwwroot` from the listener she already has** — *recommended*. A WS handshake is an HTTP GET with an `Upgrade` header, so request lines are already being parsed. One source of truth. Costs: the remote key must gate GETs too, and the page's CSP is written for the `https://octavia.avatar` origin.
- **(B) Vendor the assets into the APK** via `WebViewAssetLoader`. No host change, real HTTPS origin for free. Cost: `wwwroot` in two repos, drifting.

### The protocol cannot carry audio upstream

The larger one. Face→host has `say` (typed text) and `sight` (one JPEG). **There is no microphone message**, and `listen` toggles the mic attached to the PC. "Hold a mic button and stream the reply" is not expressible in protocol version 1.

Escalation, not a choice: **text only** (needs nothing new, proves the whole chain) → then either **on-device recognition** sending `say` (no protocol change, but her ears then differ per face and the attention gate does not apply) or a real **`audio` message** (the right destination, the most work — her `Senses` pipeline is wired to a local NAudio device).

**Superseded 08/31/2026 — the destination is the `audio` message.** See below.

## The target changed: a peer, not a viewer *(08/31/2026)*

The ask is **a tablet used the same way the desktop is**, open at the same time as it, with her mic, camera and voice all working there. Stage 13 imagined a thin away-from-home client that mostly reads; this is a peer, and it survives none of Stage 13's simplifications.

It is now **her Stage 14**, written up in her `ROADMAP.md` — the change list is host-side and this repo is downstream of it. In brief:

- **Already works:** two faces render at once today (`FaceHub` fans out to every client in `_faces`), and `camera.js` already opens the camera *in the face* — it names a wall tablet in its own header. The camera needs no protocol change.
- **The blocker under everything: faces have no identity.** `IFaceTransport.Send` is broadcast-only and `MessageReceived` says nothing about who sent it. Fine for faces that only watch; wrong the moment one has a microphone. Everything else waits on this.
- **Her voice cannot leave the PC.** `NeuralVoice` writes to a `WaveOut`. The neat way in is `OnAudioPlayed`, which is already handed the exact PCM at the moment it reaches the sound card — tee it there and the tablet's audio is in sync with the visemes for free.
- **Audio upstream is a real refactor**, not a parameter: `WhisperRecognizer` and `MicLevelMeter` each construct their own `WaveIn` and *are* the capture device.
- **`look` is broadcast**, so with two faces both cameras open and the first answer wins — which lights the privacy marker in an empty room.
- **Echo** is the thing the network makes worse. Push-to-talk on the tablet first; always-on needs real cancellation.

**Order: identity → voice out → mic in → echo → camera targeting → turn ownership → attention gate.** Voice out before mic in, because it is smaller and it makes a tablet worth looking at before it is worth talking to.

## Decisions made

| Decision | Date | Why |
|---|---|---|
| **WebView reusing the three.js face**, not a native Compose avatar | 08/31/2026 | The face is already a web page that speaks the protocol, so the avatar, the room and the lip sync come for free and future face work lands on the phone without Android changes. It is also what made the page-serving gap visible. |
| **Separate repo**, `Octavia-Android` | 08/31/2026 | Her Stage 13 called for it; it also keeps the protocol honest. |
| **`PROTOCOL.md` is not copied here** | 08/31/2026 | Two copies of a contract is one copy and one lie waiting to happen. This repo links to hers and pins the version. |
| **Test on a physical phone first**, emulator for layout only | 08/31/2026 | The GT 730 makes a VRM in an emulator painful, and an emulator has no microphone worth testing against. See [[reference-new-pc]] equivalent notes in [[Moving To The New Machine]]. |

## Open decisions

- **Route (A) or (B)** for the face page. (A) recommended. Blocks the WebView work.
- **The GitHub remote.** Not created yet — [[Octavia]] deliberately deferred hers for a day too, so this is consistent rather than an oversight. `gh` is authed as `N3cTr0`.
## Test devices *(decided 08/31/2026)*

| | Galaxy J7 Pro (SM-J730) | Xiaomi 11T Pro |
|---|---|---|
| Role | The tester, first | The destination |
| Android | **9 (API 28)** | 11 or later |
| SoC | Exynos 7870 — 8× Cortex-A53, **no big cores** | Snapdragon 888 |
| GPU | **Mali-T830, single core** | Adreno 660 |
| RAM | 3 GB | 8 GB+ |

**`minSdk 28`** — the floor of the oldest device actually exercised, which also removes the compatibility branching around `network_security_config` (cleartext to `10.1.1.x` has been blocked by default since API 28).

**The J7 Pro is a good tester because it is slow.** Transport, pairing, reconnection, WireGuard routing, the firewall rule, battery and push-to-talk all get a hostile environment, and a `subscribe`/`skip` policy that works here works anywhere.

**But it cannot judge the WebView renderer decision.** A VRM of 28 materials at up to 2048×2048 plus the room shader, on a single-core Mali-T830 with 3 GB of RAM, will not run acceptably — a fact about a 2017 handset, not about the approach. **The face waits for the 11T Pro**; everything else is proven on the J7 first. The app must be useful and testable with no face at all, which Stage 1 already required.

**Push-to-talk is now a consequence rather than a preference** — `AcousticEchoCanceler` is per-device and unreliable, and is not a plan on this SoC.

## Toolchain — and it needs no elevation *(08/31/2026)*

**The elevated session turned out to be unnecessary.** Elevation was `False`, so user-scope installs were *tried* rather than assumed, and everything that matters went in without a UAC prompt:

| Tool | Version | Notes |
|---|---|---|
| Android SDK Platform-Tools (`adb`) | 37.0.1 | `winget --scope user` |
| Microsoft OpenJDK 17 | 17.0.10 | `%LOCALAPPDATA%\Programs\Microsoft\jdk-17.0.10.7-hotspot` |
| scrcpy | 4.1 | mirror/control the device from the PC |
| **Google `android` CLI** | 1.0.15985488 | the one that changed the plan |

**Android Studio is not needed.** Google's unified `android` CLI does `create`, `sdk`, `emulator`, `run`, `install`, `screen` and `layout` — everything Studio was on the list for. Pass `--no-metrics` as a *global* flag (before the subcommand); it reports usage data otherwise. The SDK installs per-user to `%LOCALAPPDATA%\Android\Sdk`.

Install Studio anyway if the visual layout editor is wanted, but nothing is blocked without it.

**Temurin 17 was the only refusal** — no user-scope installer exists for it. OpenJDK covers it, so it was dropped rather than escalated to an elevated run.

`gh` 2.98.0 is at `C:\Program Files\GitHub CLI\gh.exe`, **not on PATH** — call it by full path. Authed by OAuth, not the PAT noted as expiring.

**Note:** `winget` writes the new PATH to the registry, and a shell started before that will not see it. Rebuild PATH from `[Environment]::GetEnvironmentVariable('Path','Machine'/'User')` in the same call, or the tools look missing when they are not.

## Core notes

- [[Overview]] — what the app is (mirror of `README.md`)
- [[Architecture]] — the four files, and which decisions are load-bearing
- [[Build & Release]] — the toolchain, building, `adb reverse`, and the vault scripts
- [[Conventions & Security Model]] — versioning, the two credentials, and the cleartext decision
- [[Roadmap]] — the stages (mirror of `ROADMAP.md`)
- [[Changelog]] — full version history (mirror of `versions.md`)
- [[Lessons Learned]] — the expensive ones, so we never pay twice
- [[Restore From Snapshot]] — rebuilding this repo if the machine is gone

## Code

- [[_Code Index]] — snapshot of every source file, regenerated by `tools\sync-vault.ps1`

## Elsewhere

- [[Octavia]] — the parent project hub, and the place to start
- [[Face Protocol]] — the host↔face contract these two repos share, and it lives in **her** repo
- [[The Host-Face Bridge]] — the socket, the virtual origins, and why loopback was reachable
- [[Moving To The New Machine]] — why elevation and execution policy behave as they do here
