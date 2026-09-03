---
project: Octavia
tags: [octavia, feature]
---

# Diagnostics

*Stage 4, v0.5.0.* How she explains herself on a machine nobody here can touch.

## The problem

Every silent failure in this project so far was diagnosed from **this** PC, with a debugger and a test harness: the microphone that opened successfully and delivered digital silence, the `<think>` filter that truncated short replies, the model that would not load. On someone else's computer there is none of that, and *"it doesn't work"* is the entire bug report.

The through-line of the stage: **every silent failure becomes a visible one.** The Stage 1 silence watchdog was the first instance; this generalises it.

## Structured logging

`octavia.log` had no levels, no rotation, and grew forever.

- Levels: `debug` / `info` / `warn` / `error`. Errors carry the whole exception — a message without a stack trace is exactly what you wish you had when the report arrives from elsewhere.
- **One file per day since v0.45.0** — `octavia-2026-09-03.log` — and *nothing schedules that*. The day is spliced into the name, so `Log.Today` is simply a different answer after midnight and the rotation happens as a consequence of writing. A server that was asleep at the turn of the day rotates on its first line of the new one exactly as a server that was awake. **When a requirement names a time, ask whether the time is the trigger or just the boundary.**
- **`LogKeepDays` (14) deletes the rest**, once per day, on the first line written. The purge reads **the file's own timestamp rather than a date out of its name** — which costs nothing and quietly clears the `octavia.log` and `octavia.1.log` every earlier version left behind. `0` keeps everything, which is what somebody chasing a fault across a month types, and must not be read as "keep nothing".
- Still rolls at **1 MB** *within* a day. Kept and demoted rather than replaced: a day is a good unit for finding things and no unit at all for bounding size, and the day something goes wrong at three in the morning is the day it writes ten gigabytes.
- The last 300 lines are kept **in memory**, so the Health panel works even when the disk does not.
- `LogLevel` and `LogKeepDays` in config.json, or in [[Her Controls]]; `OCTAVIA_LOG` redirects the whole scheme (the test harness uses it so a check can exercise a fortnight of rotation without touching the real one).
- **Notices are logged too.** They are the things she thought were worth interrupting for, and by the time a bundle arrives the one that mattered has long since faded off the screen.

## Crash handling

`DispatcherUnhandledException` used to log and swallow, which hid exactly the faults worth seeing. Now all three routes are covered:

| Route | What happens |
|---|---|
| UI thread | Logged with stack trace, **shown on her face**, still handled — she is a companion, not a batch job |
| Background thread | Logged; nothing can be done about it |
| Unobserved task | Logged as a warning and observed |

And `Task.Forget(what)` replaces every `_ = SomeAsync()`. A discarded task swallows its exception until the garbage collector happens to notice — which is precisely how a subsystem stops working without saying anything. See [[Lessons Learned]].

## The self-test

Reachable from the Health panel or by sending `selfTest`. Thirteen checks, each one there because that failure has already happened once:

| Check | Answers |
|---|---|
| Data folder | Can she write settings, models and logs at all? |
| Settings | Which profile is live — see [[Profiles & Configuration]] |
| Face transport | Did the socket bind, and is anything attached? |
| Renderer | Did the WebGL scene build? |
| Microphone | Does Windows' **own** meter see signal on the default device? |
| Speech model | Is the Whisper file downloaded, and is Silero present? |
| Voice | Is the configured voice actually installed here? |
| Avatar | Is the chosen `.vrm` where she was told it would be? |
| Camera | Whether she may open one at all — off is a pass, not a warning |
| Attention gate | Which model judges, and a loud failure when it differs from the brain's |
| Music | Switched off, no output device, or listening — the three causes of "she never dances" |
| **Rounds** | How far through the learning she is, when she last walked, and what came of it |
| Brain | Local endpoint reachable and model listed; for Claude, only whether a key is stored |

> **The Rounds row exists because the answer is almost always "nothing".** A round that finds nothing and a round that has silently stopped running feel identical from outside — hour after hour of her saying nothing — so the panel a person actually opens has to say *when she last looked*. Green either way: an empty route and a quiet night are both correct outcomes, not faults. The one amber case is a finding she never managed to say. See [[Her Rounds]].

Two deliberate properties:

- **Every failing check carries a `fix` sentence.** A red line that does not say what to try is barely better than no line. The microphone check names the RDP client setting by its full path.
- **It is free.** The local brain is pinged; Claude is *never* called. A self-test that quietly spends money is a bad self-test.

A bundle taken while she is stopped omits the transport and renderer checks entirely rather than reporting them red — nothing to say about something that was never started.

## The bundle

One zip, saved wherever the person using her chooses:

```
README.txt    what is inside, and what to check before sending it
report.txt    self-test result, system facts, recent log
config.json   her settings, with anything key-shaped removed
logs/         every log file that survives — the last LogKeepDays of them
```

Reachable from the face (**Health → Save**) *and from the tray*, because the moment you most need a bundle is the moment the face is what broke.

### Privacy, decided rather than inherited

The bundle is the thing that leaves the building, so its contents are a deliberate list rather than "whatever was lying around".

- **The log contains transcripts of things said in the room.** README.txt says so in plain language, names the file to open first, and says the bundle is still useful with lines deleted.
- File paths contain the Windows account name; the readme says that too.
- The **API key is never in it** — it stays DPAPI-sealed outside the bundle and is never written to the log. `config.json` is copied through a redactor that blanks any property named like a key, token, secret or password, so the guarantee survives someone adding a setting later.

The redactor matches **whole words** of a setting's name and only ever blanks a *string*. The first version matched substrings and blanked `Hotkey` and `MaxTokens` — over-redaction is not the safe default here; it quietly destroys the thing the bundle exists to carry.

### Taking one when she will not start

```
Octavia.exe --diagnostics C:\Users\you\Desktop\octavia.zip
```

No window, no session — it runs **before the single-instance check** on purpose. The machine, the settings and the logs are still what explain why she will not start.

## The Health panel

A drawer beside the transcript: the checks with their remedies, this machine's facts (versions, WebView2, elevation, audio device inventory, models, attachments), and the recent log. Opening it runs the test — asking for the answer as well as the question saves a click at the exact moment someone is already frustrated.

## Protocol additions

Still version 1, all additions. See [[Face Protocol]].

- **Face to host:** `selfTest`, `saveDiagnostics`, `openDataFolder`
- **Host to face:** `diagnostics { running, checks[], facts[], log[] }`, `diagnosticsSaved { path }`

`saveDiagnostics` asks for a bundle but **cannot say where**. The host writes it into `data\diagnostics\` under a timestamped name and answers with the path — a face that could name the destination would be a face that could write a file containing the log anywhere on the machine.

> **The dialog could not survive v0.26.0, and the fix is better than what it replaced.** A `SaveFileDialog` needs a dispatcher and somebody looking at it, and the host may now be running where there is neither — so **the one control that exists for when she is broken would have been broken by moving her**, silently, on the machine where it mattered. It writes to a known place instead and sends the path back over the socket, which also means whoever asked is told even from another room. The old dialog could only ever put the bundle where the person at *that* machine chose, which is no use when the machine that needs diagnosing is somewhere else.
>
> It stays host-only. There is no longer a technical reason for it to be — but the bundle carries her log, her config and a system report, and widening the authority table is a decision of its own rather than a side effect of moving a file dialog. See [[A Server, And Clients]].

## What building it caught

Two faults of exactly the kind the stage exists to expose, both perfectly silent:

- The save dialog was **constructed on a socket thread**. A file dialog is a UI object, so it threw — into a discarded task, where the exception sat unobserved. The button did nothing at all, and nothing anywhere said why. *(Both of that dialog's problems went away in v0.26.0, when the dialog did.)*
- The headless bundle blocked the dispatcher thread on a task whose continuations were posted back to that same thread, and **hung forever**.

Both are in [[Lessons Learned]].
