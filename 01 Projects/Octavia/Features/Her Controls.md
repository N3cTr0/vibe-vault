---
project: Octavia
tags: [octavia, feature]
---

# Her Controls

*`Octavia.Control.exe`, the fourth project — v0.46.0.*

A tray icon for **her server**, and a window holding everything about it that is a setting.

## Why it is a separate process

**A Windows service has no desktop.**

She runs in session 0. A tray icon drawn from inside her would be drawn where nobody can see it — no error, no missing icon, just nothing in the notification area. So the thing a person clicks has to live in *their* session and reach the service from outside.

That is the whole reason, and it is worth stating plainly because *"the server needs a tray icon"* sounds like a change to the server and cannot be one. See [[Lessons Learned]].

## What it does

| Tray | Start her · Stop her · Restart her · Settings · Open data folder · **Quit these controls** |
|---|---|
| **Brain** | Which brain, local model and endpoint, Claude model, reply length, the Anthropic key |
| **Ears** | Speech model, language, compute, wake phrase and threshold, load-on-start |
| **Voice and face** | Speaking rate, avatar, room hour, status readout, camera |
| **Rounds** | On/off, minutes between checks, **days spent learning**, quiet hours — and how far through the learning she is |
| **Integrations** | Per server: enabled, every value it is given, and a Store/Clear per sealed password |
| **Server and logs** | Profile, face port, remote access, log level, days of logs kept |

**"Quit these controls" is worded that way deliberately.** A tray whose Quit stopped a service meant to survive a reboot is a trap, and the wording is the whole guard — *Stop her* is three lines above it.

## It edits the file, not a running session

`config.json` and the sealed secret store are the two things the server reads at startup, so writing them is the whole of configuring her.

**That is what keeps it working while she is stopped** — which is exactly when somebody needs to change the setting that is stopping her. Talking to a live session is the obvious design and fails at the only moment that matters.

`OctaviaConfig.Save` does the careful part: only changed keys, written into the *base* of the file, so a profile overlay is never baked in. That was a real regression once, and the merge exists because of it. See [[Profiles & Configuration]].

> **A profile overriding a field is said out loud**, at the top of the window, naming which keys. A value edited here that the active profile also sets is written and then quietly ignored at startup — **a control that appears to work and does nothing is the worst thing a settings screen can contain**, and this one had four of them by construction.

Restarting is **stop, wait, start** rather than one command: the service control manager returns from a stop when the service says it stopped, and her unwind — three MCP children, a voice engine, 1.6 GB of Whisper — finishes after that. Starting immediately finds the port still held, which reads as *"she would not start"* and is really *"she had not finished stopping"*.

## The client stopped controlling her

> *"Remove the start stop from the client as the clients should not be able to configure server side things, they should only be able to set what they send out."*

A client is a renderer with a microphone. What it sends and what it draws are its business; the lifetime of a service on another machine is not, and the two usually sharing a box is a coincidence of this setup rather than a licence. See [[A Server, And Clients]].

**`LocalServer.Ensure` stays.** Attaching to a server, and starting one when there is none to attach to, is what makes double-clicking her work — that is not configuration, and removing it would break the ordinary case to satisfy a tidy rule.

`ServiceInstalled` and `ControlService` were **deleted rather than moved wholesale**: the part two processes now need went to `Core.ServerControl`, and the part only a client does stayed. Two copies of *"where is `Octavia.Server.exe`"* would have been two answers the first time somebody rearranged the folders.

## Checked

`SplitChecks` holds the controls to the same rule as the client — no session, no brain, no ears, no voice. **It is easier to break there**: that app exists to configure her, so the shortest path to any new setting will always look like constructing the thing that owns it.

> The check asserting the client no longer calls `ControlService` first went red on **the comment explaining that it had been removed**. Comments are stripped before anything is searched now. See [[Lessons Learned]].
