---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: ROADMAP.md
---

# ROADMAP.md

```markdown
# Octavia for Android — Roadmap

Her `ROADMAP.md` Stage 13 ends with a warning worth repeating at the top of this one:

> None of it is the app. An Android client is a WebSocket, a microphone and a renderer; the
> work that makes it possible lives in this repo and in the network, and doing it in the
> wrong order produces a phone app that can only be used on the sofa it was built on.

Stage 13's steps 1–4 are **done**, in her v0.14.0: the socket can bind the LAN, `remote.key`
survives a restart, `subscribe` lets a phone decline sixty visemes a second, and the network
decision is WireGuard on the UDM SE, written down.

Then two more prerequisites turned up on 08/31/2026, when the renderer question was actually
answered. Both live in *her* repo. Neither was visible until someone said the word "WebView".

---

## Stage 0 — The two things that are not in this repo

### 0a. The face page is not reachable from anywhere — ✅ **closed in her v0.20.0**

**Route (A) was chosen and is built.** Her socket now answers plain GETs from `wwwroot` at
`/` and her avatars folder at `/avatars/`, sharing one port and one origin with the
WebSocket. The credential moves to an `HttpOnly` cookie after the first request, because a
page's sub-resources cannot carry a query string. Verified in a browser beside her own
WebView2 face: two faces at once, the VRM off `/avatars/`, textures loading, her character
on screen. See `PROTOCOL.md` → *Serving the face*, and her `versions.md` 0.20.0.

**One consequence lands squarely on this repo:** `getUserMedia` does **not** run on a plain
`http://<lan-ip>` origin — it is not a secure context. So the tablet's camera and microphone
cannot live in the WebView. **This app owns them natively and answers `sight` itself**; the
WebView stays a renderer. That is a better fit for the protocol anyway — the *app* is the
face, and the WebView is only how it draws.

The original write-up follows, for the reasoning.



`MainWindow.xaml.cs:58` serves `wwwroot` to the built-in face with
`CoreWebView2.SetVirtualHostNameToFolderMapping`, mapping `https://octavia.avatar` to a
local folder. That is a **WebView2 feature**, not a server. There is no HTTP listener in the
process at all — `WebSocketFaceServer` is a raw `TcpListener` that only ever speaks the
WebSocket upgrade, chosen that way in Stage 3 to avoid needing a urlacl and elevation.

So Stage 13 step 1 opened the *socket* to the LAN. It did not make the *page* reachable, and
an Android WebView has no equivalent of a virtual host mapping.

**Two ways out.**

**(A) Serve `wwwroot` over HTTP from the socket she already has.** *Recommended.* A WebSocket
handshake is an HTTP GET with an `Upgrade` header, so the listener is already parsing request
lines — answering the ones that are not upgrades with a static file is a modest addition
rather than a new subsystem. One source of truth, no drift, and the promise that every future
face change lands on the phone for free actually holds.

The honest costs: the remote key has to gate GETs as well as upgrades, or the room and the
avatar are readable by anything on the LAN; and the page's CSP is written for the
`https://octavia.avatar` origin, so it needs revisiting when the origin changes. Note also
that the "this turns a private socket into a listening service" line was already crossed in
v0.14.0 — this does not cross a new one.

**(B) Vendor the face assets into the APK** and load them through `WebViewAssetLoader` at
`https://appassets.androidplatform.net/`. No host change at all, works with the socket alone,
and gets a real HTTPS origin for free — which sidesteps the cleartext problem in 0c below.
The cost is duplication: `wwwroot` would live in two repos and drift, and a sync script is a
worse version of the guarantee that (A) gives structurally.

**(B) does not actually avoid the problem — found 08/31/2026.** There are *two* virtual host
mappings, not one. `MainWindow.xaml.cs:58` maps `wwwroot`, and `:63` maps her **avatars
folder** — `OctaviaSession.AvatarUrl()` hands the face `https://octavia.avatar/<file>` and
`hello` carries it as a URL.

A VRM is user data. It lives in her git-ignored `data\` folder, it is chosen at runtime from
the `avatars[]` list, and it is not in either repo — so it **cannot** be baked into an APK.
Route (B) therefore still needs the host to serve files over HTTP; it just serves fewer of
them, while adding a sync script and a second mechanism to keep in step.

Since (B) pays route (A)'s main cost anyway and then adds drift on top, **(A) is the
recommendation and (B) is close to ruled out.** Worth stating plainly rather than leaving the
choice open on a false symmetry.

**Not urgent, though.** The J7 Pro cannot judge the renderer anyway (see *The test devices*),
and Stage 1 needs no page at all. This can be settled while Stage 1 is being built.

### 0b. The protocol cannot carry audio upstream

This is the larger one. The face→host table in `PROTOCOL.md` has `say` (the user *typed*
something) and `sight` (one base64 JPEG, answering `look`). **There is no message that
carries microphone audio.** `listen` toggles *her* microphone — the one plugged into the PC,
in the room with the PC.

So "hold a mic button and stream the reply", the sentence Stage 13 step 5 is built around, is
not expressible in protocol version 1. Discovered 08/31/2026, before any Kotlin, which is the
entire reason step 5 was put last.

**Three ways out, and they are a sensible escalation rather than a choice.**

1. **Text only.** The app types, she answers, the transcript streams back. Needs *nothing*
   new — `say`, `turn` and `caption` already exist. It proves the transport, the pairing, the
   VPN and the renderer end to end, which is most of the risk in this project, and it is
   genuinely useful for "how is everything at home?".
2. **Recognise on the phone.** Android's on-device `SpeechRecognizer` produces text; send it
   as `say`. No protocol change. The cost is that her ears become different depending on
   which face you are talking to — the VAD, the hallucination defences and the attention gate
   she has in `Senses/` do not apply, and a cloud recogniser would put her conversation
   through Google, which is a privacy decision and not just a technical one.
3. **Add audio upstream to the protocol.** An `audio` message carrying PCM frames, so the
   phone is a microphone and her existing ears do the work — which is what "a renderer plus a
   microphone, not a second Octavia" actually means. This is the right destination and the
   most work: her `Senses` pipeline is wired to a local NAudio device, and feeding it a remote
   stream is a real change, not a parameter. It is a protocol version 1 addition (new message
   type, which the versioning rules explicitly allow) and belongs in `PROTOCOL.md` before
   anything here depends on it.

**Decided 08/31/2026: the destination is 3.** The ask is not Stage 13's away-from-home client
that mostly reads — it is a **tablet that is a peer to the desktop face**, open at the same
time as it, with her mic, camera and voice working there as they do here. That is her
**Stage 14**, written up in her `ROADMAP.md`, and it makes option 2 a dead end: on-device
recognition would give her different ears in every room and bypass the attention gate she
already has.

Ship 1 anyway, because it proves the transport, the pairing and the VPN while the host-side
work happens — but build it knowing 3 is where it goes.

### 0c. Cleartext, if we go with (A)

`http://10.1.1.x:<port>` is cleartext, which Android has blocked by default since API 28.
This needs a `network_security_config.xml` scoped to the LAN subnet — **not**
`usesCleartextTraffic="true"` across the whole app. Over WireGuard the traffic is encrypted
on the wire regardless, so this is about satisfying the platform, not about the actual threat.
Route (B) avoids it entirely for the page, though the WebSocket still needs it.

---

## The test devices *(decided 08/31/2026)*

| | Galaxy J7 Pro (SM-J730) | Xiaomi 11T Pro |
|---|---|---|
| Role | The tester, first | The destination |
| Android | **9 (API 28)** | 11 or later |
| SoC | Exynos 7870 — 8× Cortex-A53, **no big cores** | Snapdragon 888 |
| GPU | **Mali-T830, single core** | Adreno 660 |
| RAM | 3 GB | 8 GB+ |

**`minSdk 28`.** It is the floor of the oldest device we actually test on, there is no reason
to claim support below what is exercised, and it removes a great deal of compatibility
branching — including around `network_security_config`, which is the thing that makes
cleartext to `10.1.1.x` work at all (blocked by default since API 28).

### What the J7 Pro is good for, and what it will lie to you about

It is an **excellent** tester for almost everything in this project, precisely because it is
slow. Transport, pairing, reconnection, WireGuard routing, the firewall rule, battery
behaviour, push-to-talk, and the audio path all get a genuinely hostile environment — and a
`subscribe`/`skip` policy that works here works everywhere. If sixty visemes a second is ever
going to be a problem, it will be a problem on this device first, which is useful.

**It cannot tell you whether the WebView renderer decision was right.** A VRM with 28
materials at up to 2048×2048, plus the room shader, on a single-core Mali-T830 with 3 GB of
RAM and eight A53s and no big core, will not run acceptably — and that is a fact about this
handset, not about the approach. Judging route (A)/(B) or the whole WebView choice on it
would be reading the wrong instrument.

**So: the face renders on the 11T Pro, and the verdict waits for it.** Everything else gets
proven on the J7 Pro first. Plan the WebView work to be turned off — the app should be
useful, and testable, with no face at all, which Stage 1 already requires.

### Consequences to design around now

- **Push-to-talk stops being a preference.** Android's `AcousticEchoCanceler` is per-device
  and unreliable at the best of times; on a 2017 mid-range Exynos it is not a plan.
- **No NDK for now.** The app is pure Kotlin/JVM, so the ABI never comes up. That changes the
  day real echo cancellation is wanted — WebRTC's AEC would need `arm64-v8a` *and*
  `armeabi-v7a` to cover this device.
- **The WebView is updatable through Play**, so Chromium is likely current even though the OS
  is not. WebGL2 should be *present*. Present and usable are different questions.
- **Compose is fine on API 28**, and is still the right choice — the transcript and controls
  are not what will be slow here.

## Stage 1 — It connects and it says something — ✅ **done in 0.3.0**

Proven on the real J7 Pro: she answered a typed question from the handset and both turns came
back, with her log reading `face connected over socket (2 attached)`. The development loop is
**`adb reverse tcp:8848 tcp:8848`**, which points the handset's loopback at the host's over
USB — so her socket stays on `127.0.0.1`, `RemoteAccess` stays off, and no firewall rule or
elevation is involved. Wireguard replaces only the address.

**Hardened and closed in 0.4.0.** The gap named here — untested across a locked screen, a
doze or a lost transport — has been tested on the J7 Pro and two real faults came out of it:
a 401 was being retried forever against a credential that could never work, and the socket
was held for the life of the ViewModel rather than while she was on screen. Both fixed; the
measurements are in `versions.md`.

One deliberate limit remains, and it is a limit rather than a bug: **this is a foreground
client.** With the screen off it holds nothing, so she cannot reach the phone unprompted.
That becomes a foreground service with a notification when a stage needs it — a visible
thing rather than a socket left running by accident.

The original scope follows.



The smallest thing that proves the whole chain: pair, connect, type, get an answer.

- Settings: host address, port, remote key. Stored in `EncryptedSharedPreferences`.
- The socket client: connect, `ready`, receive `hello`, send `subscribe` with
  `skip: ["viseme", "level"]`, reconnect with backoff when the phone sleeps or roams.
- A Compose screen: her state, the transcript from `turn`, captions from `caption`,
  `overheard` shown faintly — the protocol says show it, never swallow it.
- A text field sending `say`.
- **No WebView yet.** No avatar, no room, no microphone.

This is deliberately the boring version, and it is where the real risks are: WireGuard
routing, the firewall rule, key normalisation, the socket surviving a locked screen.

## Stage 2 — Her face

The WebView, once stage 0a is settled.

- Load the face page, pass the token, let the existing renderer do everything it already does.
- Drop the `subscribe` skips while the face is visible — the avatar needs `viseme` and `level`
  to be worth looking at — and reinstate them when it is not.
- Expect a performance pass. A VRM at 60 fps is a different proposition on a handset than on
  a desktop, and the room shader is not free.
- Tear the WebView down when it is not on screen. This is a battery, not a desktop.

## Stage 3 — A peer, not a viewer

Follows her Stage 14 and cannot start before it. **Item 1 of that stage — faces having any
identity at all — is specified** in the vault at `01 Projects\Octavia\Stage 14 - Face
Identity.md`, written from this side because this is the client that needs it. It is a change
to *her* repo; nothing here moves until it lands. What lands here:

- **Her voice out of the tablet speaker** — a host→face `audio` message, teed from
  `NeuralVoice.OnAudioPlayed` so it is already in sync with the visemes.
- **Push-to-talk**, streaming 16 kHz mono 16-bit PCM up. Resample on the tablet; she wants
  what Silero and Whisper want. Push-to-talk rather than always-on because a speaker and an
  open mic across a network is an echo problem that real cancellation has to solve, and that
  is its own piece of work.
- **The camera**, which needs nothing new here — `camera.js` already opens it in the face,
  and it was written for a wall tablet by name. It needs *her* to send `look` to one face
  instead of broadcasting it.
- **Both open at once**, which already renders correctly today and will need testing for the
  things that are not rendering: whose turn it is, whose camera opened, where the voice went.

## Stage 4 — The house

`hello` gains whatever Stage 12 gives it, and this app shows it. Not designed yet, because
Stage 12 is not built yet and designing a client for an interface that does not exist is how
you get a client that has to be rewritten.

---

## Standing constraints

Inherited from her repo, and they apply here without amendment.

- **This app holds no API key and runs no model.** If a feature seems to need one, it belongs
  in the host.
- **A face ignores message types and fields it does not recognise** rather than failing. New
  ones may be added at any time within protocol version 1.
- **Check `protocol` in `hello`** and refuse to run against a major this app does not know.
- **Never forward her socket.** WireGuard's port, and nothing else.
```
