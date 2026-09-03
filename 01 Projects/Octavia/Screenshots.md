---
project: Octavia
tags: [octavia, screenshots]
---

# Screenshots

A record of what she looked like when each thing was checked, and **what was being checked** — not a gallery. Every one of these was taken to answer a question, and the question is the caption.

Files live in `Screenshots\`, named `v<version> - <subject>.png`. They are deliberately **not** in the `Code\` snapshot: they are evidence, not source — see [[Restore From Snapshot]].

## How they are taken

The face lives in a WebView2, so a browser screenshot is not the real thing. These are captures of the actual application window.

**Since 0.19.3 this is two committed scripts rather than scratch code**, because a set that cannot be retaken after the chrome moves is a set that goes stale silently:

```
pwsh -File tools\shoot.ps1 'what it shows'      # capture, named for the current version
pwsh -File tools\poke.ps1 1036 67               # click, in window coordinates
pwsh -File tools\poke.ps1 880 400 -Scroll -6    # wheel, for a control below the fold
```

`shoot.ps1` reads the version out of `Octavia.App.csproj`, so a shot cannot be filed under the wrong release, and sizes the window to **1100x780** first — every shot in this folder is that size, and a set taken at whatever size the window happened to be cannot be flipped through. `poke.ps1` takes window coordinates, which is exactly what you measure off the previous shot, so the two agree by construction.

Both attach to her input queue before calling `SetForegroundWindow` and then **verify the window actually came forward**, refusing to act if it did not. That is not defensive padding: `SetForegroundWindow` fails silently from a background process and the click — or the capture — then lands on whatever window *is* in front. It has cost a debugging round before. See [[Lessons Learned]].

The earlier sets opened drawers through **WebView2's accessibility tree**, because `InvokePattern` is not offered by those elements. A plain click at the right coordinates turned out to be enough and is far less to maintain.

That mattered more than expected once: the first attempt to open the Health panel clicked straight into the *Settings* drawer, which was still open and covering the button.

## When to take them

**After every version bump**, as part of the same change set as the code and the vault sync — see [[Conventions & Security Model#Vault upkeep]]. Most releases change no pixels and need nothing; the judgement is only ever "did anything visible move?". A release that changed her appearance and was never photographed cannot be reconstructed from a diff afterwards, and a changelog entry describing the chrome is no substitute for seeing it.

`tools\check-vault.ps1` reports whether the current version has any shot at all, and names the newest one if not. It **reports rather than fails** — the point is to make you look, not to demand a photograph of a bug fix.

---

## v0.26.1 — her ears open before anyone asks

![[v0.26.1 - her ears open before anyone asks.png]]

**`EARS Whisper large-v3-turbo (local)` on a fresh session, with nobody having pressed anything.** Every shot before this one said `EARS not started` until somebody asked her to listen — and v0.25.1's shot made a point of that pairing being *correct*.

This is the pairing changing, and it closes [[Roadmap]] item 11. The first press of a cold session used to lose the seconds Whisper spent loading, silently, producing a truncated sentence rather than an obvious nothing. The models load at startup now, where nobody is talking, so there is no gap to fall into rather than a mechanism for surviving one.

**It opens no microphone**, which is the distinction v0.25.1 drew and this depends on: building the recogniser touches no device, and `listen` or a held button is still what opens one. The mic button is unpressed here and her ears are open.

## v0.26.0 — the client on her server, mid-conversation

![[v0.26.0 - the client on her server, mid-conversation.png]]

**There is nothing of her in this window.** No session, no brain, no ears, no voice — it is a WebView2 pointed at `http://127.0.0.1:8848`, holding a socket like any other face. It is worth photographing precisely *because* it looks the same as v0.25.1: an architecture change that shows on screen would have been a worse one.

Her mouth is open on a viseme and the state pill says `SPEAKING`, which is the part that had to be proved rather than argued: the visemes and the PCM still leave the same buffer at the same instant, and now they cross a socket to get here.

**The turn on screen was driven by a different face entirely** — `attach-face.ps1`, in the same room — which is the other half of the claim. The client is not privileged; it is one renderer among however many are attached, and rooms still decide who sees what.

`BRAIN qwen2.5:7b-cpu (local)` and `EARS not started` are the server's facts, arriving in `hello` from another process.

## v0.26.0 — the client with her server stopped

![[v0.26.0 - the client with her server stopped.png]]

**The shot that mattered most, and the one that could not exist before this release.** While she *was* the window, a dead socket meant a dead application — there was nothing to reconnect to and nothing to say. Now the server restarts on its own, and a face that quietly stopped being connected would look exactly like one that had not.

So the bar is **persistent**, not a notice: notices fade after seven seconds, and this is a condition rather than an event. `LOST HER — RECONNECTING`, with the amber dot, and a backoff from 500 ms to 15 s underneath it.

The placard still shows her last known state, which is honest — the client is reporting what it was told, not guessing at what is true now.

Restarting the server brought this window back on its own, with a new `FaceId` and a fresh `ready`. **That re-announcement is the rule easiest to get wrong**: a reconnected face is a *new* face to the host, so a page that announced only once would find itself silently back in the host room with no camera. See [[A Server, And Clients]].

## v0.25.1 — idle, her ears shut and her microphone button offered anyway

![[v0.25.1 - idle, her ears shut and her microphone button offered anyway.png]]

**`EARS not started`, and the microphone button is there regardless.** That pairing is the whole fix. Her ears are *shut* — `ListenOnStart` is off, nothing has opened Whisper — and pressing that button is now what opens them, rather than something that requires them already open.

Until v0.25.1 the same pairing was a lie on a handset. `listen` was doing two jobs: opening this machine's microphone, which is a host-room device, and starting the **recogniser**, which is being-wide. [[One Being, Many Rooms]] locked `listen` to the host room — correctly — and took the second job with it, so a room face could never start her ears at all. `micAccepted` compounded it by reporting "already started" instead of "will accept", so the phone hid a button that would have worked.

The desktop could never show the symptom, which is the point worth remembering: this shot looks identical to the two before it, and the bug lived entirely in a face that is not this one.

## v0.25.0 — the host room, whose buttons item 10 does not touch

![[v0.25.0 - the host room, whose buttons item 10 does not touch.png]]

**Identical to the shot below it, and that is half the claim.** Item 10 gives a *remote* face its microphone and camera back by borrowing them from whatever it is embedded in. The desktop is in the host room, has no embedder, and is a secure context — so all three of its console buttons behave exactly as they did: the microphone still **toggles** `listen` rather than being held, the eye still opens this machine's camera, and the page never looks for an embedder that is not there.

The two shots differ in nothing, which is what "a browser tab behaves exactly as today" has to look like. The interesting picture is the one that cannot be taken here — the handset, where the same two buttons are back and the microphone is press-and-hold.

**`PROFILE cloud` and a green `BRAIN claude-sonnet-5` again**, for the reason set out in v0.24.1: it is worth re-photographing a fact the record got wrong for four versions.

## v0.24.1 — on the cloud profile, with the key that was always there

![[v0.24.1 - on the cloud profile, with the key that was always there.png]]

**This shot exists to falsify a sentence.** For four versions the record said there was no API key on this machine, and that claim held Stage 14's camera test open and half-excused deferring the tool loop. The placard reads `BRAIN claude-sonnet-5` on `PROFILE cloud`, and — the part that actually settles it — **the dot beside it is green**. That dot is `hasKey`, which is `SecretStore.ReadApiKey()` returning a non-empty string, so the key is not merely present as a file but decrypts under this account. It cost nothing to check and no call was made. See [[Changelog]] 0.24.1.

Compare it with the v0.24.0 shot above: same window, `BRAIN qwen2.5:7b-cpu (local)` there against `claude-sonnet-5` here. **That is the entire difference, and it was a command-line flag.**

**Three buttons instead of two.** The eye is back, because `Camera` is `true` in config — which is itself worth reading. Her log for the day names the room on every change:

```
12:10:40  warn   camera enabled in room 'phone'
12:13:24  info   camera disabled in room 'phone'
12:47:45  warn   camera enabled in room 'phone'
14:02:15  warn   camera enabled in room 'host'
```

Every `phone` line is in-memory only and died with the process; only the `host` one reached `config.json`. That is the per-room rule from v0.24.0 behaving exactly as designed, visible in the log without anybody instrumenting it — and it is also why **the camera is currently switched on at the desk**. It was a deliberate act at 14:02 and has been left rather than quietly undone, but by this project's own standard — off by default, the only sense that is — it should be turned off when nobody is testing.

## v0.24.0 — idle, with a phone in its own room

![[v0.24.0 - idle, with a phone in its own room.png]]

**The interesting thing about this shot is what it does not show.** A real handset was attached from `10.1.1.181` at the moment of capture, in room `phone`, with both of its connections — the native client and its WebView panel — in that room together. The desktop shows no sign of it, which is the whole of item 9: before this release a phone was a second window onto *this* screen.

Read three things off it. The status placard says `PROFILE dev` with **no room after it** — the room is appended only when it is not the host's, so a `?room=phne` typo is visible rather than mysterious. The microphone button is still there, because this face *is* the host room and may drive this machine; on the handset that button and the four host-only Settings rows are hidden, and the host refuses the messages behind them regardless. And `EARS not started` is the resting state with `ListenOnStart` off, not a fault.

> **The phone arrived on its own, three seconds after she started, already sending `room: "phone"`.** The Android side had implemented its half against the spec before the host understood the field — until this build, `ready.room` was simply ignored. Acceptance criterion 8 was confirmed on real hardware without anybody arranging it. See [[One Being, Many Rooms]].

## v0.23.1 — the remote key fix

![[v0.23.1 - idle on 0.23.1, the remote key fix.png]]

**Nothing moved, and that is the point of taking it.** A one-character authentication fix should change no pixels, and this confirms it changed none: idle, the stats placard reading `PROFILE home` and `BRAIN qwen2.5:7b-cpu (local)`, both console buttons where they were.

Two things are worth reading *out* of it rather than into it. `EARS not started` — `ListenOnStart` is off, so this is the resting state, not a fault. And still **no eye button**, because `Camera` went back to `false` after being switched on and off again; see the correction in [[Eyes]].

## v0.21.2 — the camera gets a switch

![[v0.21.2 - the camera finally has a switch in Settings.png]]

**Checking:** that the two new rows sit correctly in the **off** state, which is the one every fresh install sees. *Let her see you* is clear, the picker is disabled, and — the second attempt at this shot — it reads *"Whichever the browser picks"* rather than being **blank**. The first version only populated the menu when the camera was on, and a disabled empty select reads as broken rather than as switched off. Worth the retake.

Note what is *not* in the console bottom-left: no eye button, because the sense is off. That is the whole reason this was reported as the camera icon having disappeared. See [[Eyes]].

---

## v0.21.1 — she gets an icon

### Her own mark in the title bar

![[v0.21.1 - her own icon in the title bar, at last.png]]

**Checking:** that the icon actually reached the window, which is the one thing
`<ApplicationIcon>` does not prove on its own — WPF falls back to the exe icon only when the
window does not set its own, and this confirms the fallback works rather than assuming it.
Before this the title bar showed the generic .NET box. See [[Branding]].

Incidentally caught: `MUSIC 155 bpm`, headphones on, and the room tinted purple. She was
hearing something play while the shot was taken — none of which was arranged.

---

## v0.19.3 — local-first, and the chrome as it settled

The first set taken with [[#How they are taken|the committed scripts]], and the first to include her **window** rather than only her face — the title bar is in frame because it is part of what she looks like.

### Idle, with the room to herself

![[v0.19.3 - idle, the placard faded and the room hers.png]]

**Checking:** the 0.19.1 fix. Nine seconds after the last word the caption fades and she keeps the whole stage — and, crucially, **does not change size doing it**. The placard used to be a sibling below the stage, so hiding it grew the stage, fired the canvas `ResizeObserver` and re-framed the camera; she visibly jumped twice a cycle. Overlaid, the viewport is a constant. See [[The Room]].

Also the answer to the question that started this release: `PROFILE home` and `BRAIN qwen2.5:7b-cpu (local)`. Before 0.19.3 this said `live` and `claude`, with no key stored, so every turn was refused.

### The typing field, staying put

![[v0.19.3 - the typing field, which now stays open between messages.png]]

**Checking:** that the field opens focused with the button showing its active state. Until 0.19.2 sending a line closed it, so a typed conversation meant re-opening it for every message.

### Answering

![[v0.19.3 - answering, the caption over the room.png]]

**Checking:** two fixes in one frame, which is why this is the shot worth keeping. The caption sits **over** the room as a subtitle on its scrim rather than in a strip beneath it (0.19.1), and the field has stayed open, cleared and focused after sending (0.19.2). The Hush square is present because she is thinking, and absent everywhere else in this set.

### The drawer, on a real turn

![[v0.19.3 - the drawer on Transcript, a real local-brain turn.png]]

**Checking:** the local brain actually answering through the whole stack — not a probe, a turn. Her reply is *"I can't see the house yet"*, which is both in character and true: [[Roadmap]] stage 12 is the one that gives her the house.

### Settings, both halves

![[v0.19.3 - settings, appearance through the devices she uses.png]]

**Checking:** appearance, speech, voice and the device pickers, each reading back from the host rather than defaulting in the page. *Hears what you play* names the actual endpoint it is listening to.

![[v0.19.3 - settings, the status readout switch and the API key out of her face.png]]

**Checking:** the three things that moved here. **Show the status readout** switches off the panel over her top-left corner; **Speech recognition runs on** is the CPU/GPU choice added when this machine turned out to have the better processor; and the **API key** now lives down here instead of nagging from a pill on her face. The hint says it is not needed on a local brain, which is now the default.

---

## v0.15.0 — dancing to what the machine plays

![[v0.15.0 - dancing to loopback, headphones on.png]]

**Checking:** headphones on and a tempo in the status strip from audio heard through WASAPI loopback. The tempo is right this time — the VM's flattened endpoint that made v0.8.0 read 109 for a 132 bpm track was a decoder bug, not the hardware. See [[Music]].

---

## v0.12.0 — she has a face at last

![[v0.12.0 - textures loading, the CSP fix.png]]

**Checking:** textures rendering, for the first time in the project's life. Every model had always come up blank-faced, and the cause was neither the lighting nor a missing `KTX2Loader` — the CSP simply did not list `blob:`, and three.js decodes embedded glTF textures to a `Blob` and loads them from a `blob:` URL. Every texture in every model was being blocked regardless of format. 20 of 28 materials textured, zero console errors.

---

## v0.9.2 — watching

### She follows you

![[v0.9.2 - watching, camera pill and eye button.png]]

**Checking:** the whole watching contract in one frame — the red eye button lit beside the microphone, the standing **CAMERA** pill beside the state pill, and her head turned toward movement the camera saw. Compared against a shot seconds earlier her yaw has visibly changed, and with `holdGaze` suppressing her own saccades that movement can only have come from the tracker. See [[Eyes]].

---

## v0.8.1 — the dev panel

### Driving her by hand

![[v0.8.1 - the dev panel driving her.png]]

**Checking:** that the panel actually reaches the renderer, and that two independent things hold at once — `surprised` set by hand while `Hold the face` keeps the host from wiping it, and the headphones worn with no music playing at all. Both are states that were previously only reachable by *causing* them.

### Everything below the fold

![[v0.8.1 - props, room and senses.png]]

**Checking:** the sections that scroll off — props, the room's hours, and the Senses row. Senses is the one group whose buttons leave the renderer, because a microphone and a loopback are devices the face does not own; the footer says so, since the distinction is invisible otherwise.

---

## v0.8.0 — Stage 7, music

### Headphones on, from music she actually heard

![[v0.8.0 - headphones on, heard through the loopback.png]]

**Checking:** the whole chain in one frame — a track played through the speakers, heard back through WASAPI loopback, decided to be music, and answered by putting headphones on the head bone of a character the prop knew nothing about. The status strip carries the tempo, which is what makes "she is not dancing" and "there is nothing to dance to" different sentences.

**The tempo is wrong and that is the honest part.** It reads 109 for a 132 bpm track, because this VM's audio endpoint flattens everything to full scale. The same analyser locks to within 0.3 bpm offline at every tempo tested. See [[Music]].

### The switch, and what it promises

![[v0.8.0 - the music switch in settings.png]]

**Checking:** that `hello` carries the setting and the face reflects it — the switch is on because the host said so, not because the page defaulted that way. The hint under it is doing real work: it is the only place a person is told that nothing is recorded.

### Where "she never dances" gets answered

![[v0.8.0 - output device and music in the report.png]]

**Checking:** `Default output` and `Music listening` reaching the system report. Between them they separate the three causes that look identical from the sofa — switched off, no output device, or nothing playing.

---

## v0.7.0 — Stage 6, the neural voice

### The health panel, live

![[v0.7.0 - health panel, live self-test.png]]

**Checking:** that the self-test reports the engine she is *actually* speaking with. Before the fix it constructed a `SapiVoice` regardless and would have cheerfully reported a Windows voice while Piper was talking. Here it reads `Piper (Jenny Dioco (en-GB, medium))`.

Also visible: the microphone failure carrying its full remedy, which is the whole point of [[Diagnostics]] — a red line that does not say what to try is barely better than no line.

### The settings drawer

![[v0.7.0 - settings drawer.png]]

**Checking:** appearance, speech engine, voice and room lighting all reachable in one place, each with a hint. `Speech: Neural voice` and `Voice: Jenny Dioco` are both being read back from the host rather than assumed by the page.

### Her, with a voice

![[v0.7.0 - neural voice active.png]]

**Checking:** she boots straight into the neural engine from the saved setting — note `VOICE Jenny Dioco (en-GB, medium)` in the status row, where a Windows voice used to be.

---

## v0.6.1 — the settings menu

### The menu driving the real face

![[v0.6.1 - settings menu driving the real face.png]]

**Checking:** that a settings change made from an *external* face reaches the built-in one. Appearance was switched to the plaster bust and lighting to dusk; this is the WebView2 window following both. It is the protocol from [[The Host-Face Bridge]] doing exactly what it was built for.

### Dusk on the VRM

![[v0.6.1 - dusk lighting on the VRM.png]]

**Checking:** the full-day lighting cycle from [[The Room]] applying to a character rather than the bust — wall, key light and rim all moving together.

### The VRM as her default

![[v0.6.1 - VRM as the default face.png]]

**Checking:** the saved `AvatarFile` loading at startup, and the new **Settings** button sitting in the console row.

---

## v0.6.0 — Stage 5, the face and the room

### First VRM, posed and framed

![[v0.6.0 - first VRM, idle pose and portrait framing.png]]

**Checking:** two fixes at once. Every VRM is authored in a **T-pose** — the format defines a rest position, not an idle — so her arms are posed down on load. And the camera frames her from her own head bone, so any character ends up the same size on screen instead of being scaled to fit the bust's framing. See [[The Avatar Interface]].

The capture caught a terminal window overlapping at the left edge; that is the screen capture, not the app.

### three.js r180, and a room

![[v0.6.0 - three.js r180, bust in the new room.png]]

**Checking:** the 2021 UMD build replaced with r180 as ES modules without the bust losing anything — and the flat wall replaced by the shader environment. This is the shot that confirmed the vignette was finally rendering: the previous version used `smoothstep` with its edges reversed, which is undefined in GLSL and silently drew nothing.

---

## The 09/02–09/03 run *(v0.35.0 → v0.46.0)*

Filed under `Screenshots\`. The ones that carry something a diff cannot:

| Shot | What it settles |
|---|---|
| `v0.39.1 - her mouth moves again, mid-vowel` | The bug that only a photograph could confirm: her voice played perfectly and her mouth never moved, because a 20 ms clock made every chunk smaller than a viseme frame |
| `v0.40.0 - speaking in her new voice, mid-sentence` | Kokoro `af_heart`, mouth moving, caption up — the engine replaced with nothing downstream noticing. See [[The Voice]] |
| `v0.41.0 - she asks before cutting power to a port` | *"Do you want to power cycle port 4 on the UDM?"* and then **stopping**. The confirmation gate, live, against the real gateway. See [[Hands]] |
| `v0.42.0 - the Rounds row, saying she has nothing to walk yet` | The row that exists so a week of deliberate silence is not mistaken for a broken job. See [[Her Rounds]] |
| `v0.43.0 - she runs; the UniFi password is still to be stored` | A declared secret with nothing behind it yet — left **unset** rather than handed `""`, which would have reported a login failure and sent somebody to check the account |
| `v0.43.1 - the sealed UniFi password reaches her tool server` | The same row after `--secret`, proving the DPAPI value survives the spawn into a child process |
| `v0.44.0 - a rehearsal of an hourly finding, said out loud` | What she sounds like when she finds something, without waiting for her to find something |
| `v0.45.0 - a log file per day, and quiet from midnight to 08-30` | Rotation with nothing scheduled — the day is spliced into the filename — and the quiet window the owner asked for |
| `v0.46.0 - her server's own settings window` | [[Her Controls]], and the header warning that the active profile overrides four of the fields below it |
| `v0.47.0 - one executable again, her tray and settings` | The merge that undid `Octavia.Control` after a single release: same binary, different mode |
| `v0.48.0 - the key badge, and no secret on show` | **"Stored", in green, above the fold.** The state was always detected and always below the fold, which is why the report was *"it doesn't say"* |
| `v0.48.0 - the UniFi key sealed, not on show` | `UNIFI_API_KEY` as a masked box with a badge, beside `UNIFI_HOST` still in plain text — the distinction the window now draws by itself |
| `v0.49.0 - her face at v0.49.0, the release that taught her to check` | Nothing visible changed, and filing it says so. The release is two faults sharing one sentence — a consent rule that never granted, and a tool that reported success without ever looking |
| `v0.49.1 - her face at v0.49.1, and a yes that finally lands` | The hour after v0.49.0, and the same nothing to see. A yes now runs the call **she described out loud** rather than the one the model writes next — proven on `home`, the local brain she is actually started under |
| `v0.49.2 - her face at v0.49.2, told what she can actually do` | The third invisible release in a row, and the cause was the least visible thing of all: a system prompt that had told her for six versions she had **no hands**, so she narrated switching a port on and never called anything |
| `v0.50.0 - her face at v0.50.0, with the firewall in reach` | Thirteen tools now. She can read the firewall — 66 rules answered as six zones and a catch-all — reboot a device, and put a client on or off the network. The firewall is the one she can only read |
| `v0.50.1 - three lines at the bottom, holding where she started` | The caption capped to three lines, so a six-sentence answer no longer covers her. It holds where she **started** rather than where she is — the following shipped, was watched failing, and was reverted the same evening |
| `v0.51.0 - mid-sentence, on the line she is actually saying` | Eight sentences composed in one burst, and the caption sitting on Mars and Venus with Mercury scrolled above and the next line faded. **Where her voice is, not where the writing got to** — the engine marks each utterance's audio and the `Pacer` says how far it has been heard |
| `v0.51.0 - the room to herself, with a version in the corner` | The same thing after three refinements: the status readout off by default, `0.51.0 · local` at 28% in the bottom right, and the line she is on sharp between two faded ones |

> **Two of these are photographs of a refusal**, and that is not a coincidence. The confirmation gate and the learning silence are both features whose correct behaviour is *not acting*, and neither leaves a trace in a diff or a log line that reads as success.

## Gaps

**Stages 1–4** have no screenshots. Those were verified from logs, harness output and the protocol rather than by eye, which was correct for what they were — but it does mean there is no visual record of her before the room.

**v0.10.0 and v0.11.0 are the expensive gap.** That is the whole console rebuild: three hand-written drawers replaced by one with tabs, the tokens introduced, the API key moved out of the status strip, Hush made transient, the chrome put into the room's own light. The largest visual change the project has had, and nothing was captured while it happened. v0.12.0 is the next shot after it, so the before-and-after is missing its before.

**v0.13.0–v0.14.x and v0.16.0–v0.19.2** are also unphotographed, though less painfully — 0.19.3 shows where that run of changes ended up, even if not how it got there.

This is the gap that [[#When to take them|the after-every-bump rule]] exists to stop reopening.

**A different kind of gap has opened since.** From v0.10.0 to v0.38.0 the shots were *taken* — the rule held — but this note stopped describing them, so there are roughly thirty images in `Screenshots\` with nothing here saying what they settle. The filenames carry a caption each, and `check-vault.ps1` counts them per version, so nothing is lost; what is missing is the sentence explaining why each one was worth taking. **Taking the photograph and writing down what it proves are two habits, and only the first one stuck.**

> **And it reopened immediately after being written down.** v0.43.0 through v0.47.0 were photographed and then went undescribed here for five releases, while this paragraph sat at the bottom of the note complaining about exactly that. Backfilled in v0.48.0. **A lesson written in the same note as the lapse does not prevent the lapse** — the table is the artefact that has to be updated, and noticing it is missing is not something a person does while looking at a different file.

## Photographing the settings window

`shoot.ps1` looks for `Octavia`, the client. Her **settings window belongs to `Octavia.Server`**, so for two releases the shots of it were captured by hand while a committed script sat next to them claiming to be how it is done.

It takes `-Settings` now: server window first, and `-KeepSize` implied, because the settings window has its own shape and the 1100×780 the face is standardised at would stretch it into a window that never existed. Her face stays comparable shot to shot; the settings window is photographed as it is.

```
pwsh -File tools\shoot.ps1 'the key badge, and no secret on show' -Settings
```

A tray with **no window open** has no `MainWindowHandle` and is indistinguishable from nothing running, so the failure message names both ways in.
