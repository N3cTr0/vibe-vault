---
project: Octavia
tags: [octavia, deepdive]
---

# The Host-Face Bridge

How the being reaches the renderer — and why this small piece decides how far she can go.

*Stage 3 landed in v0.4.0 — the protocol is now written down in the repo's `PROTOCOL.md`, and the socket is the primary transport.*

## Today

`FaceHub` implements `IFaceTransport` and fans every message out to two channels: the built-in page's `WebViewFaceTransport`, and `WebSocketFaceServer` broadcasting to every attached socket face. Inbound messages from either are merged. The session pushes a message and does not know how many renderers are listening.

**The built-in page prefers the socket too**, so it is no longer a special case — one code path, one protocol. postMessage survives only as the fallback for when the port cannot bind.

### The question that decided the design

Could a page served from the virtual `https://octavia.face` origin open `ws://127.0.0.1`? A secure context talking to an insecure scheme is normally mixed content and blocked.

**It can.** Loopback is treated as *potentially trustworthy*, so mixed-content blocking does not apply. Confirmed empirically — `face connected over socket (1 attached)` in the log on first run. Had it failed, the design would have had to keep the embedded page permanently on postMessage and treat the socket as an extra; instead the transports unify.

### The server

Raw `TcpListener` plus `WebSocket.CreateFromStream`, not `HttpListener`. `HttpListener` needs a urlacl reservation for anything outside the default namespace, which would mean elevation or a setup step; the handshake is a dozen lines and `CreateFromStream` handles the framing, so nothing hand-rolled matters.

Security is loopback binding plus a per-run token compared in fixed time. A bad or missing token is refused at the HTTP handshake with `401` and never becomes a WebSocket. The token is logged at startup so an external face can be pointed at her, and changes every run. **It is a speed bump, not a boundary** — it stops a stray local page driving her, not a local attacker who can read the log.

### A bug worth remembering

The server originally abandoned the socket on receiving a close frame. A face that disconnected *politely* got an EOF error on its way out, because the server never sent its half of the close handshake. Found by the protocol test, not by the app.

Two details that matter on the WebView2 channel:

- **Dispatcher marshalling.** Audio callbacks, SAPI viseme events and brain streaming all arrive on worker threads; `PostWebMessageAsJson` must be called on the UI thread. `Send` checks `CheckAccess()` and re-posts if needed. Without this, viseme events — the highest-frequency message — would tear the app down.
- **Nothing is trusted.** A malformed message from the page is logged and dropped, never thrown.

## The virtual origin

`SetVirtualHostNameToFolderMapping("octavia.face", wwwroot, Allow)` and then `Navigate("https://octavia.face/index.html")` — not `file://`.

That makes the page a **secure context**, which is required for `getUserMedia` when the camera arrives in Stage 9, and avoids `file://` origin restrictions on scripts and fetches. It costs nothing now and is awkward to retrofit later.

A second virtual host was added in v0.6.0 for the same reason: `https://octavia.avatar` maps read-only to her avatars folder, so the face can load a VRM the host chose without ever being handed a filesystem path.

Confirmed the hard way: loading `index.html` directly from disk in a normal browser renders it as a static snapshot with scripts inert — which is also why the face's own error reporting exists.

## The renderer has no console

The host cannot see the page's devtools output in the log, so the page reports on itself:

- `window.onerror` and `unhandledrejection` are forwarded as `faceError` messages into `octavia.log`.
- The `ready` message carries `faceBuilt: typeof window.Face === 'object'` — true only if the WebGL context and the entire scene graph constructed without throwing.

That one boolean is how a headless launch verifies the renderer works. `face ready; scene built` in the log means three.js is alive.

## Why this stage mattered

Anything that speaks the protocol is now a legal face:

- the WebView2 page
- a browser on a wall tablet
- **an Unreal application rendering a MetaHuman** (Stage 8)

That last one is the point. Photorealism becomes *attaching a different renderer* instead of rewriting the application. The protocol is the durable asset; the bust is disposable. See [[Roadmap]] and [[Architecture]].

**Now checked, not just demonstrated.** `tools\attach-face.ps1 -Conformance` drives a running host through a turn, a self-test and a forget, and reports which host-to-face messages arrived and whether each carried the fields the protocol promises. That turns "anything that speaks the protocol is a legal face" from a claim into a test — and it found a gap immediately: a face attaching mid-session was never told her current expression, because `emotion` is only sent on change. `hello` now carries `state`, `emotion` and `emotionWeight`. See [[The Photoreal Decision]].

**Proven, not assumed.** `tools\attach-face.ps1` attaches to a running Octavia as an external face — a PowerShell script, nothing to do with WebView2 — and drives her. A full session on 08/30/2026: `hello protocol 1`, then a typed message, then `caption`, `turn`, `state thinking`, her reply, and viseme messages streaming as she spoke it. Several faces can attach at once, which is exactly what makes developing a new renderer beside the old one practical.

## The protocol as it stands

Full reference in the repo's `PROTOCOL.md`.

**Host → face:** `hello`, `state`, `level`, `viseme`, `caption`, `turn`, `notice`, `needKey`, `cleared`.

**Face → host:** `ready`, `say`, `listen`, `hush`, `forget`, `setKey`, `setVoice`, `faceError`.

Stage 4 added the diagnostics messages — `selfTest`, `saveDiagnostics` and `openDataFolder` in, `diagnostics` and `diagnosticsSaved` out (see [[Diagnostics]]). Stage 5 added `emotion` and the settings messages; Stage 6 added `setVoiceEngine`; Stage 7 added `music` and `setMusic`. All are pure additions within version 1 — a face must ignore what it does not recognise, which is what keeps that true.

### `music`, and why a beat travels alone

`music` is the first message with **two shapes**: `{ beat: true }` on its own, and `{ beat: false, playing, bpm, energy }` for the state. They are deliberately not merged.

A beat is worthless late. Coalescing it with a state update — which is rate-limited to about twelve a second, like `level` — would delay it by up to eighty milliseconds, and a nod eighty milliseconds behind the music is visibly wrong in a way a stale energy value never is. So the beat goes out the instant it is found and carries nothing else.

The corollary is a rule for face authors: **move on `beat`, never on a clock of your own running at `bpm`.** The host is the one listening. A face predicting its own beats drifts out of time within a few bars, and has no way to notice.

**No audio crosses the protocol and none is kept.** The host analyses the output mix locally and sends three numbers. See [[Music]].

One line of the contract is worth stating on its own: **`saveDiagnostics` asks for a bundle but cannot say where it goes.** The host raises its own file dialog and a person picks the path. A face that could name the destination would be a face that could write a file containing the log anywhere on the machine.

## The security boundary

The face's CSP is `default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data:` with `connect-src` narrowed to loopback WebSocket origins only. It still cannot reach the wider network. The API key travels *in* via `setKey` and never comes back out — `hello` carries a boolean.

The socket widened the attack surface deliberately and consciously: a local WebSocket is reachable by anything on the machine, so loopback binding and the handshake token were part of the stage's design rather than an afterthought. Both refusal cases are covered by tests.
