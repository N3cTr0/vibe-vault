---
project: Octavia
tags: [octavia, feature]
---

# Her Controls

*A tray icon and a settings window for her server. Its own executable in v0.46.0; **merged back into the server in v0.47.0**.*

## One executable, three modes

> *"I don't see why we should have the server and the control app which control the server, this should be merged into 1 app... the server is her backbone."*

| No arguments | The tray: install her as a service, start, stop, restart, settings |
|---|---|
| `--settings` | The tray with the window already open |
| `--console` | Run her here and print — the old no-argument behaviour |
| `--service` | The service itself, in session 0, no UI |

**No arguments used to mean *run her and print*.** That is still `--console`, and still what `dotnet run` and the shortcuts get — but it was the wrong default for the thing a person double-clicks, where what they want is to see whether she is running and be able to change it.

### The reasoning that split them was half an argument

**A Windows service has no desktop.** She runs in session 0, so an icon drawn from inside her would be drawn where nobody can see it — no error, no missing icon, just nothing in the notification area.

That is true, and it never required a second **binary**. It required a second **mode**: the tray is always launched by a person in *their* session, and the service never draws anything. See [[Lessons Learned]].

**The tray manages the service rather than hosting her.** Two ways of being running would mean two things for the settings window to restart and a race over the port.

## What the window holds

| Brain | Which brain, local model and endpoint, Claude model, reply length, the Anthropic key |
|---|---|
| Ears | Speech model, language, compute, wake phrase and threshold, load-on-start |
| Voice and face | Speaking rate, avatar, room hour, status readout, camera |
| Rounds | On/off, minutes between checks, **days spent learning**, quiet hours — and how far through the learning she is |
| Integrations | Per server: enabled, every value it is given, and a Store/Clear per sealed password |
| Server and logs | Profile, face port, remote access, log level, days of logs kept |

**"Quit this tray" is worded that way deliberately.** A tray whose Quit stopped a service meant to survive a reboot is a trap, and the wording is the whole guard — *Stop her* is four lines above it.

## It edits the file, not a running session

`config.json` and the sealed secret store are the two things the server reads at startup, so writing them is the whole of configuring her.

**That is what keeps it working while she is stopped** — which is exactly when somebody needs to change the setting that is stopping her. Talking to a live session is the obvious design and fails at the only moment that matters.

`OctaviaConfig.Save` does the careful part: only changed keys, written into the *base* of the file, so a profile overlay is never baked in. See [[Profiles & Configuration]].

> **A profile overriding a field is said out loud**, at the top of the window, naming which keys. A value edited there that the active profile also sets is written and then quietly ignored at startup — **a control that appears to work and does nothing is the worst thing a settings screen can contain**.

Restarting is **stop, wait, start** rather than one command: the service control manager returns from a stop when the service says it stopped, and her unwind — three MCP children, a voice engine, 1.6 GB of Whisper — finishes after that. Starting immediately finds the port still held, which reads as *"she would not start"* and is really *"she had not finished stopping"*.

## Three faults from the merge, and only one was in the plan

- **`WinExe` was tried first**, to stop a console flashing on a double-click, and it silently breaks every switch a person types: a shell does not wait for a windowed process, so output races the prompt, the exit code is lost, and `--secret` has no console to read a keypress from. She stays a console application and the *tray mode* calls `FreeConsole`.
- **`[STAThread]` went missing.** A WPF application never has to think about it — its generated entry point carries the attribute — and a console `Main` does not. Startup is clean, the tray works, and the **first window** throws, after the console has been freed.
- **A refused second instance held a modal dialog**, so it stayed alive holding it. It prints and exits now, and the first launch shows a balloon: the other half of refusing quietly is that the launch which *worked* has to be discoverable.

## What the merge cost

[[Diagnostics|`SplitChecks`]] asserted that the settings UI never constructs a session, and the assembly boundary did that for free. There is no boundary now, and the server legitimately constructs her.

The check is **file-scoped instead, and weaker on purpose rather than by accident**: it names `Tray.cs` and `SettingsWindow.xaml.cs`, and adds `Being.Start` to what they may not contain. The pressure is real — that window exists to configure her, so the shortest path to any new setting will always look like reaching for the thing that owns it.

## The client stopped controlling her

> *"Remove the start stop from the client as the clients should not be able to configure server side things, they should only be able to set what they send out."*

A client is a renderer with a microphone. What it sends and what it draws are its business; the lifetime of a service on another machine is not. See [[A Server, And Clients]].

**`LocalServer.Ensure` stays.** Attaching to a server, and starting one when there is none to attach to, is what makes double-clicking her work — that is not configuration.
