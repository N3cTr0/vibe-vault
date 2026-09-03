---
project: Octavia
tags: [octavia]
---

# Overview

**Octavia** is a desktop companion: you talk, she answers out loud, and a face on screen moves while she does it. She is being built toward an always-on agent that hears the room, reacts to music, sees through a camera, and eventually controls the house — with a photoreal animated face.

She began as a single HTML file (`C:\Projects\talking-avatar.html`) — a three.js plaster bust using the browser's Web Speech API and calling the Anthropic API straight from the page. That prototype proved the look and killed itself on three limits at once, which is why she is a desktop app now. See [[Architecture]].

## Tech stack

| Thing | Choice |
|---|---|
| Host | `Octavia.Server.exe`, a headless console app on .NET 10 (`net10.0-windows10.0.19041.0`) — see [[A Server, And Clients]] |
| Clients | `Octavia.exe` (a WPF window around a WebView2), [[Octavia Android]], or any browser. None of them contain her |
| Face | three.js r180 (ES modules), served over HTTP by the server itself |
| Avatar | A VRM character via `@pixiv/three-vrm`, or the plaster bust — see [[The Avatar Interface]] |
| Room | A shader environment with a full day of lighting — see [[The Room]] |
| Brain | Any OpenAI-compatible local server, streamed — **the default**; or the Anthropic SDK (`claude-sonnet-5`) |
| Ears | Silero VAD (ONNX) → Whisper (`Whisper.net`, CPU + CUDA runtimes) |
| Voice | **Kokoro `af_heart`**, out of process, and there is no other — chosen by ear out of twenty-two, see [[The Voice]] |
| Lip sync | Read out of the waveform. No engine has ever given her phoneme timings that survived |
| Attention | A small local model judges what was addressed to her — see [[The Attention Gate]] |
| Eyes | One still from the face's camera, only when a question needs it — see [[Eyes]] |
| Music | Beat detection, locally — but the **server holds no device** since v0.38.0, so a client must report what is playing, and nothing does yet. See [[Music]] |
| On her own | `Watchman` walks a route on a clock; she learns what normal is for a week before saying anything — see [[Her Rounds]] |
| Audio | NAudio for capture and playback; a small in-house FFT for analysis |
| Hands | An MCP client over stdio, so an integration is a server rather than a branch — see [[Roadmap]] stage 12 |
| Serving | The face socket also answers GETs, so a phone or tablet can load her face from the host itself — see [[Face Protocol]] |
| Rooms | A face belongs to a *space*, and a conversation on a phone is not the one at the desk — see [[One Being, Many Rooms]] |
| Secrets | DPAPI, sealed to the current Windows account — the API key, and a password per tool server, never in `config.json`. See [[Conventions & Security Model]] |
| Controls | `Octavia.Control.exe` — a tray icon and a settings window for the server, in the user's session because a service has no desktop. See [[Her Controls]] |
| Distribution | **Four** exes plus one `wwwroot` beside them — see [[Build & Release]] |

## Why she is not a web page

The prototype could not become the thing she is meant to be. Three hard blockers, not preferences:

- **System audio capture.** Reacting to music needs WASAPI loopback. A browser's only route is `getDisplayMedia`, which re-prompts with a screen-picker every session.
- **Always-on listening.** Background tabs get throttled; microphone permission does not persist for `file://`; there is no tray, no autostart, no global hotkey.
- **A LAN Home Assistant.** A page cannot call `http://homeassistant.local:8123` — mixed content and CORS.

Plus the one already biting: the prototype put the API key in the page and called the API from the client.

What survived the move is the face itself, and it survived *because* it was kept dumb. See [[The Face]].

> **All three blockers are the *server's* problem, and none of them is the window's** — which is why v0.26.0 could take the window away entirely and lose nothing. The face went back to being a page loaded over HTTP, which is what it was in the prototype; what changed is where the thinking happens and who holds the key. See [[A Server, And Clients]].

## What she does today

Talk or type; she answers in two or three short sentences, spoken aloud in a neural voice, with a mouth that moves to the sound she is making and an expression read from what she is saying. She hears you through Whisper running locally — nothing acoustic leaves the machine. She stands in a room whose light follows the time of day, as a VRM character or the plaster bust, and a settings drawer changes any of it while she is running. Put music on and she hears it through the machine's own output, puts headphones on and moves to the beat — locally, without a model call.

Leave her listening and she no longer answers everything she hears: a small local model judges what was addressed to her, for free, and what she declines is shown faintly rather than swallowed — see [[The Attention Gate]]. Ask her something that needs eyes and she takes a single still, if you have turned that on — see [[Eyes]].

Music playing on *another* device in the same room reaches her too, through the microphone rather than the machine's output — off by default, because lyrics are speech and the attention gate has to survive them. See [[Music]].

She can also explain herself: a self-test with remedies, and one zip you can send from a machine nobody can reach — see [[Diagnostics]].

Her face is no longer only hers. The socket that carries the protocol now serves the page itself, so a phone or a wall tablet can load her face from the host over the LAN rather than needing a copy of it — which is what an Android client is being built against. See [[Face Protocol]] and [[Roadmap]] stage 13.

**She cannot yet act on the house.** The seam is built and tested — an MCP client, a risk policy, a confirmation rule for anything irreversible — and configured servers show up in the status readout, so the plumbing is real and visible. What is missing is the brain-side loop that lets her *call* one. She has no wake word either. Both are stages, not omissions — see [[Roadmap]].

## Where things live

| Path | What |
|---|---|
| `C:\Projects\Octavia` | The repo |
| `src\Octavia.App\` | The host: `Brain/`, `Senses/` (with `Music/`), `Voice/`, `Audio/`, `Diagnostics/`, `Face/`, `Core/` |
| `src\Octavia.App\wwwroot\` | The face: the loop, the room, the avatars, the console |
| `src\Octavia.App\wwwroot\lib\` | Vendored three.js and three-vrm — excluded from the vault snapshot |
| `tools\EarsTest\` | Headless test harness and probes — see [[The Ears]] |
| `tools\*.ps1` | Vault sync and check, the face dev server, an external face, a mock MCP server, and the screenshot pair — see [[Screenshots]] |
| **`data\`** | Config, log, DPAPI-sealed key, Whisper models, `avatars\`, `voices\` — **inside the repo since v0.11.0**, git-ignored, so one folder is the whole of her. `%APPDATA%\Octavia` only for an installed copy; see [[Profiles & Configuration]] |
