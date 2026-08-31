---
project: Octavia Android
tags: [octavia, octavia-android, conventions]
---

# Conventions & Security Model

> How this repo is written and what it does with secrets. Her equivalent is [[Conventions & Security Model|Octavia's]], and where they differ it is noted rather than assumed.

## Versioning

Pre-release `0.x`. **PATCH by default** (`0.x.y`) for fixes, tweaks and copy changes; **MINOR** (`0.x.0`) only for a new subsystem or a notable behaviour change. Roadmap stages are substantial by definition, so a completed stage takes a MINOR.

Headers in `versions.md` use ISO `YYYY-MM-DD` — an internal doc convention. **Displayed dates are `MM/DD/YYYY`.**

The version lives in the newest `versions.md` header and nowhere else; both vault scripts read it from there, so the file name of a screenshot cannot drift from the release it documents.

## Code

- **Comments carry the *why*.** Rationale belongs in the source when it would otherwise be lost, and in the commit message when it is about the change rather than the code. Density matches her repo: sparse, but never absent where a decision was made.
- **`PROTOCOL.md` is not copied here.** It lives in her repo; this one links to it and pins the version it was written against. Two copies of a contract is one copy and one lie waiting to happen.
- **Ignore what you do not recognise.** Every inbound message is read defensively. See [[Architecture]].
- **`gradlew` is pinned to LF** in `.gitattributes`. Git's autocrlf would otherwise store the wrapper script with CRLF, which fails as a bad interpreter on Linux, macOS and CI.

## The security model

**The VPN is the boundary, not the key.** Her socket is meant to be reachable only over the UDM SE's Wireguard. Her socket is *never* forwarded; the one forwarded port is Wireguard's own UDP port, which does not answer unauthenticated packets at all and so is silent to a scanner.

### The two credentials

| | Source | Where it works |
|---|---|---|
| **Per-run token** | Her log, regenerated every start | **Loopback only** — including an `adb reverse` tunnel |
| **Remote key** | Settings on the host, survives restarts, revoked by rolling | Anywhere, and only while `RemoteAccess` is on |

The token is deliberately refused from anything but loopback: it is written to her log and carried in a URL, and it is scoped to a process on that machine.

### Where the credential is stored here

Plain app-private `SharedPreferences`. This is a deliberate match to her own decision rather than a second, worse one: `RemoteKey.cs` stores `remote.key` **in the clear** on the host, with a comment saying why — it has to be *readable* to be shown in Settings and typed into a phone, which rules out sealing it.

The honest limit: this protects against other apps, not against someone holding an unlocked phone. Rolling the key on the host invalidates every paired device, which is the revocation story at both ends.

### Cleartext

`network_security_config.xml` permits cleartext, and the file says why at length. In short: her socket speaks plain HTTP and `ws`, the deployment is behind Wireguard where the wire is encrypted regardless, and the host is typed in by the user so there is no domain list to scope to. **TLS on her listener is the real fix and belongs in her repo** — at which point that file gets tightened.

### What this app never does

No API key. No model. No conversation state. No decision about what she says. If a feature seems to need one of those, it belongs in the host — see [[Architecture]].

## Links

- [[Octavia Android]] · [[Architecture]] · [[Build & Release]] · [[Lessons Learned]]
- [[Face Protocol]] — the contract, in her repo
