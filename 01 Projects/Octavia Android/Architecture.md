---
project: Octavia Android
tags: [octavia, octavia-android, architecture]
---

# Architecture

> How the phone client is put together, and which decisions are load-bearing. The parent architecture is [[Architecture|Octavia's]]; this note only covers what is different on a handset.

## The one rule

**This app is a renderer.** It holds no API key, runs no model, owns no conversation and decides nothing about what she says. Every fact it displays arrived as a JSON message over a WebSocket. If a class in here ever starts *choosing*, the logic belongs in the host instead.

That is not a preference invented for the phone — it is the contract in [[Face Protocol]], written in her Stage 3 precisely so a face could one day be somewhere else.

## The shape

```
MainActivity
└── FaceViewModel          holds only what she has said
    ├── FaceSocket         the only thing that talks to her
    └── Settings           host, port, credential
        └── FaceScreen     state pill, transcript, caption, text field
```

Four files, and the smallness is the point: everything interesting is in her repo.

## The message lists are deliberately not repeated here

[[Face Protocol]] is regenerated from her `PROTOCOL.md` on every vault sync, so it cannot drift. A hand-kept copy can, and in her vault two of them did — [[Architecture|her Architecture note]] and [[The Face]] sat five face→host messages and eight `hello` fields behind, one of them under the words "that is the entire contract".

**This client is the reason that mattered.** It was written against the gap, guessed `value` where `say` wanted `text`, and she accepted it and silently did nothing. Naming a specific message while making an argument is fine; reproducing the table is not. Link to the mirror.

## Why `org.json` and not a generated parser

The protocol says a face must **ignore types and fields it does not recognise** rather than failing, because new ones may be added at any time within version 1. Defensive `opt*` reads express that directly. A schema-bound parser expresses the opposite — it turns "she added a field" into "the app crashes", which is exactly backwards for a client that will always be older than the host it talks to.

`FaceViewModel.onMessage` is therefore a `when` with no complaining `else`. An unrecognised message type is not an error; it is simply not used yet.

## Why OkHttp is the only third-party dependency

Android has no WebSocket client of its own. Hand-rolling RFC 6455 to avoid one well-known library would be a poor trade, and everything else — JSON, preferences, UI — is on the platform or in AndroidX.

## The transport, and the two credentials

She accepts two different secrets and treats them very differently:

| | Where it comes from | Where it works |
|---|---|---|
| **Per-run token** | Her log, regenerated every start | **Loopback only** — including an `adb reverse` tunnel |
| **Remote key** | Settings on the host, survives restarts | Anywhere, and only while `RemoteAccess` is on |

`FaceSocket.credentialParam` tells them apart by shape — the token is 32 hex characters, the remote key is four groups of five from an alphabet with no `0`/`O` or `1`/`I`. They go in different query parameters, so guessing wrong is a refused connection rather than a subtle bug.

## What the WebView will and will not do, when it arrives

The face page is served by her socket over HTTP ([[Roadmap]] stage 0a, her v0.20.0), so a WebView can load it. But **`getUserMedia` does not run on a plain `http://<lan-ip>` origin** — that is not a secure context.

So the camera and microphone are **native to this app**, and the app answers `sight` itself. The WebView is only how she is drawn. That is a better fit for the protocol than the arrangement it replaces: the *app* is the face, and the renderer inside it is an implementation detail.

## Links

- [[Octavia Android]] — the hub
- [[Octavia]] — the parent project
- [[Face Protocol]] — the contract, which lives in her repo
- [[Build & Release]] — the toolchain and the development loop
- [[Conventions & Security Model]]
