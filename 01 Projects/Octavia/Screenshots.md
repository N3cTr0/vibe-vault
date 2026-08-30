---
project: Octavia
tags: [octavia, screenshots]
---

# Screenshots

A record of what she looked like when each thing was checked, and **what was being checked** — not a gallery. Every one of these was taken to answer a question, and the question is the caption.

Files live in `Screenshots\`, named `v<version> - <subject>.png`. They are deliberately **not** in the `Code\` snapshot: they are evidence, not source — see [[Restore From Snapshot]].

## How they are taken

The face lives in a WebView2, so a browser screenshot is not the real thing. These are captures of the actual application window:

- `tools`-adjacent scratch scripts capture the window by handle (`GetWindowRect` + `CopyFromScreen`).
- The drawers are opened by finding the button in **WebView2's accessibility tree** and clicking its bounding rectangle. `InvokePattern` is not offered by those elements, so a real click at the right coordinates is the way in.

That mattered more than expected: the first attempt to open the Health panel clicked straight into the *Settings* drawer, which was still open and covering the button.

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

## Gaps

No screenshots exist for stages 1–4. Those stages were verified from logs, harness output and the protocol rather than by eye, which was correct for what they were — but it does mean there is no visual record of her before the room.
