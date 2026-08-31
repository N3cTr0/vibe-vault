---
project: Octavia
tags: [octavia, code]
source-path: versions.md
---

# versions.md

```markdown
# Octavia — Version History

Pre-release `0.x` scheme. **PATCH by default** (`0.x.y`) for fixes, tweaks and copy
changes; MINOR (`0.x.0`) only for a new subsystem or a notable behaviour change.
Roadmap stages are substantial by definition, so each completed stage takes a MINOR.

Headers use ISO `YYYY-MM-DD` — an internal doc convention, separate from displayed
dates, which are `MM/DD/YYYY`.

---

## 0.19.3 — 2026-08-31

**Four roadmap bugs closed, one of which was live.** MINOR-adjacent but PATCH: nothing
changes shape, but her default brain does.

**She was one shortcut away from being mute.** The config on this machine said
`Profile: "live"` → `Brain: "claude"`, and no key has ever been stored — so every turn
would have hit "No API key yet. Paste one below and try again." She worked only because
the shortcut passes `--profile dev`. Started any other way, from the exe, from
`dotnet run`, from a fresh shortcut, she could not answer at all. The API-key nag that
was cluttering her face in 0.18.x was this, correctly reporting a real misconfiguration.

Stage 10's local-first profiles are now built, and the important half is not the new
names: **base `Brain` defaults to `local`**. An unnamed or misspelled profile falls back
to the base settings, and a keyless Claude is the worst possible thing to fall back to —
she looks broken rather than limited. Shipped profiles are `home` (local,
`large-v3-turbo`, the default), `cloud` (Claude) and `dev` (local, `small.en`). `live`
still resolves to what it always meant, so no existing config changes behaviour. An
undefined profile is now a **warning** that names the ones that exist; it was an info
line, which is exactly how this hid.

**Stage 10a is closed, both halves.** `EarsTest` gained `SyntaxChecks`, which takes the
option that entry preferred: load the real page in the WebView2 already on the dependency
list and let Chromium parse it. No hand-rolled parser, no Node, same engine that runs it
for real, and served from the same virtual `https://` origin so the CSP behaves as it does
in the app. It asserts no `SyntaxError`, `window.Face` published, and that the bridge would
have sent `ready`.

It was proved by breaking it. An orphan `});` appended to `bridge.js` — the v0.18.0 fault
exactly — turns it red and names the file and line, with `window.Face` still fine and
`ready` missing, which is that bug's precise signature. The host half, `WatchForFace`,
gives the face 30 seconds to say `ready` and otherwise logs an error and shows the
fallback panel: a surface that survives the renderer being dead, which is the whole point.

**Stage 2a is closed.** `LocalBrain`'s streaming loop reads to null instead of testing
`EndOfStream`, so it no longer blocks a thread pool thread waiting for the token it then
politely awaits. The build is at **zero warnings**. The audit that entry asked for came
back clean: `EndOfStream` occurred exactly once in the project.

**Docs.** `README.md` described a console that has not existed since 0.18.0 and opened by
calling her "a talking bust with Claude behind her" — both halves of that are now wrong.
It documents the drawer and its tabs, the typing button, the status-readout setting and
the local-first default. `PROTOCOL.md` turned out to already carry `camera` in both the
`hello` table and its own `look`/`sight` section, so that item was closed before it was
opened.

*A check nobody has watched fail is not a check. The syntax check was written, passed
immediately, and that proved nothing — the only informative run was the one against a
deliberately broken file.*

---

## 0.19.2 — 2026-08-31

**The typing field stays where you left it.** PATCH.

Sending a line closed the field. That was written as "the field has done its job and the
room comes back", which is a reasonable sentence and wrong about how anybody actually uses
it: a person who typed once is usually about to type again, and they were made to re-open
it for every single line. `submitTyped` no longer closes anything — it clears the box and
puts the cursor back in it.

Closing is now the keyboard button's job, or **a minute of not using it**, whichever comes
first. The timer restarts on every keystroke and every send, and it only ever fires on an
**empty** field: a half-written line is somebody still thinking, and a strip of chrome is
not worth throwing their sentence away. Escape still closes it immediately, unless she is
talking, in which case Escape stops her instead — that was already true and is unchanged.

**Roadmap gains stage 2a**: `LocalBrain.cs:96` is `while (!reader.EndOfStream)`, which is a
*synchronous* read — the CA2024 warning, and the only one in the build. On a server-sent-event
stream there is nothing to peek at until the model emits the next token, so the loop blocks a
thread pool thread waiting for the line it then politely `await`s on the next statement. It
costs more here than it used to: the brain is pinned to the CPU, tokens are slow, and the
thread is parked for the length of every reply.

*Tested by temporarily dropping the timer to 1.5s and driving the three cases — empty field
closes, draft survives, post-send field closes on its own clock — then restoring the minute.
A one-minute timeout that is only ever reasoned about is a one-minute timeout nobody has run.*

---

## 0.19.1 — 2026-08-31

**The placard fades instead of collapsing.** PATCH: fixes the rescale that came with it.

v0.19.0 gave the placard a `max-height` collapse specifically so it would not leave a band
of empty room behind. It worked, and it introduced a worse problem: the placard was a flex
sibling *below* the stage, so collapsing it grew the stage by 92px. The canvas is watched by
a `ResizeObserver`, a resize recomputes the fit and the camera distance, and the result was
that she visibly jumped size every time the caption came and went. The thing meant to give
her the room was re-framing her twice a cycle.

The fix is to stop the stage changing height at all. `#placard` now lives **inside** `#stage`
as an absolutely-positioned overlay — a subtitle over the room rather than a strip beneath
it — sitting on a short gradient scrim so the text stays legible against a bright wall, and
`pointer-events:none` so it never takes a click meant for her. The room is full height in
both states; only opacity animates.

Measured both ways: with the caption up and down, `#stage` and `#scene` both report 615px,
so the observer never fires and the camera never moves.

*The hidden-pane artifact from 0.19.0 bit again and is worth restating in its sharper form:
while the pane is hidden, `requestAnimationFrame` never fires and running transitions freeze.
A frozen transition reports a settled-looking `opacity` — 0.65 in this case — that is neither
end of the animation, and it reads exactly like a CSS rule being overridden. Test the cascade
by setting `transition:none` and reading the two states directly; that answer does not depend
on anything being rendered.*

---

## 0.19.0 — 2026-08-31

**She gets the room when nobody is talking.** MINOR: the placard comes and goes.

- **"In residence" is gone** from the header. It meant "she runs here rather than in the
  cloud" — true, unremarkable, and not worth a permanent line in the room.
- **The placard collapses after nine seconds of quiet** and the stage takes the space, so
  she is simply bigger when nothing is being said. Any caption or any state but idle brings
  it straight back.

Two details that took a moment to get right. The linger is **nine seconds, not zero**: a
reply that vanishes the instant she stops speaking is unreadable, and the caption is the
only record outside the transcript. And it **collapses rather than fades** — `opacity:0`
alone leaves a 92px band of nothing, which is the opposite of the point. `min-height:0` is
not enough either, because the caption still has an intrinsic height; it takes `max-height`
against a real number to animate down to nothing.

**Roadmap gains stage 10a**: nothing in this project checks the face's own syntax. A
JavaScript parse error is invisible to `dotnet build` and to `EarsTest`, so the build goes
green and the face is simply dead — which is how v0.18.0 shipped a broken bridge for a few
minutes. The entry proposes parsing `wwwroot/*.js` in the harness, and the more valuable
half: having the host notice that `ready` never arrived and say so.

*Two hours were lost to a measurement artifact worth writing down: **the browser pane does
not run WebGL or style recalculation while it is hidden**, so screenshots came back with an
empty scene and `getComputedStyle` returned stale values. Both looked exactly like real
bugs. Front the pane and capture in the same batch before believing either.*

---

## 0.18.0 — 2026-08-31

**The room gets quieter.** MINOR: the status readout moved and became optional.

- **The drawer button sits outboard of the state**, to the right of it. The state is about
  the room and reads first; the drawer is the way out.
- **The status readout floats over her top-left corner** on translucent glass, instead of
  taking a strip of chrome along the bottom. It is reference material — glanced at, not
  read — so it belongs *in* the room quietly rather than in a band of its own. It does not
  take clicks: a readout that swallows a click over the scene is a trap.
- **`ShowStats`** turns it off entirely, from Settings. On by default, because it answers
  most of the questions anyone asks about her; off is the setting for actually looking at
  her.
- **The missing-key pill is gone.** A permanent amber warning across the bottom of the
  room is one you stop seeing, and it nagged whoever was looking at her rather than whoever
  could act on it. Settings carries it now: the field is marked, its label reads **API key
  — needed**, and it gains an amber rule down its edge so someone who came for something
  else still finds it.

*(A `sed` deleting four lines by number took the wrong four and left an orphan `});`,
which broke the whole bridge — the drawer stopped opening. Caught by the console, not by
the build, because nothing compiles this file.)*

---

## 0.17.0 — 2026-08-31

**The console gets its corners back.** MINOR: the control surface moved.

Three changes, from a marked-up screenshot:

- **The drawer button is in the header**, beside the state it belongs with. It had been
  sitting in the console row among the microphone and the keyboard — a settings menu
  filed with the controls you reach for mid-sentence. Smaller up there, because it is a
  way out rather than something you press while talking.
- **The status readout moved to the right of the control row**, from underneath it. The
  left of that bar is where you act and the right is where you look; having both stacked
  down the left made it read as a wall of text with a microphone on top.
- **The row spans the full width** instead of a centred 860px column, which was pulling
  both ends toward the middle and leaving the corners empty.

The pills are right-*positioned* but left-*aligned* inside their block. Right-aligning the
text as well gave every label a different starting x, and a list you scan down wants one
edge to run your eye along.

**Not fixed: the headphones.** Four attempts, and adjusting by eye did not converge — each
round fixed the last complaint and introduced another. Height is right; width and depth
are not. What was learned is written into ROADMAP.md stage 11 so the next attempt starts
from it: `Box3.setFromObject` on a skinned mesh returns the *bind pose* (the body reads
±0.69 wide, arms out), the face mesh is half-width 0.109 against the 0.164 in use, and
measuring the skull alone buries the cups in a long-haired character's fringe. The missing
quantity is the hair silhouette at ear height. Worth a dev-panel slider rather than a
rebuild per guess.

---

## 0.16.2 — 2026-08-31

**The duplicate music logging, which was not what it looked like.** PATCH.

Five identical `music: N bpm` lines in one second looked like the two sources talking over
each other. It was neither source: the transition was logged on `state.Playing !=
_last.Playing`, but `_last` is only assigned **after** the 80 ms throttle returns early —
so the same change re-reported on every frame until a send finally went through and moved
`_last` on. The log now happens where `_last` is updated, so the two cannot drift apart.

Both sources do also need telling apart, so `MusicWatcher` has a `Name` — `output` for the
loopback, `room` for the microphone — and it appears in the line.

**And they were genuinely competing.** Both wrote to the same face state with no rule, so
the tempo would flicker between what this machine plays and what is in the room, with
neither reading trustworthy. The loopback now wins whenever it has something: clean
dynamics, no room, no gain control. The microphone only speaks when the loopback is silent.

---

## 0.16.1 — 2026-08-31

**The cups sit on her ears, and the room source is proven.** PATCH.

### Room music, verified

v0.16.0 shipped `MusicFromRoom` untested against real room audio. It works: with music
playing on a *different computer* and the loopback probe reading **energy 0.00**, she
reported **141 bpm at confidence 0.49** through the microphone. Both halves of that matter
— the silent loopback is what proves it is the mic, and 0.49 against loopback's 1.00 is
exactly the reduced certainty a boom mic in a room was predicted to give.

### The headphone depth, measured rather than argued

Height was right after v0.16.0; depth was not, and the sign was the reason. `vrm-avatar.js`
notes that VRM 1.0 faces -Z, which invites the assumption that "behind the head" is +Z.
**After the loader plugin has finished, it is not**: the head bone's axes come out
world-aligned and the eye bones sit at *greater* z than the head bone, so the face points
+Z and behind is negative.

Settled by shoving the whole assembly to +0.10 and watching it land in front of her nose —
one deliberately wrong value being worth more than another round of reasoning. The offset
is now 0.26 of the head half-width behind the eye line, which puts the cup on the ear
rather than the temple or the back of the skull. The comment records the measurement so
the next person does not re-derive it from a misleading premise.

---

## 0.16.0 — 2026-08-31

**She can hear a room, her headphones sit on her ears, and the dance stopped twitching.**
MINOR: a new source of hearing.

### Stage 11a — the microphone as a second ear for music

Loopback is what *this computer* plays. A speaker across the room — another PC, a phone —
never touched it, which is how "she will not dance" turned out to mean "the music was on a
different machine". `MusicFromRoom` runs a second `MusicAnalyzer` on the microphone frames
the voice detector is already reading, so it costs one subscription and no extra capture.

Off by default, and honest about two things: a boom mic in a reverberant room gives far
worse dynamics than a loopback, so expect less certainty; and it only works while her ears
are open, which is a different condition from `Music` and will surprise someone.

**Built and passing every check, but not yet verified against real room audio** — that
needs a speaker in the room, which is a thing to do rather than a thing to assert.

### The headphones sit where her ears are

They were sized and placed from head *height*, an assumption about width taken from an
unrelated measurement. They now come from the **eye bones**, which VRM 1.0 requires: the
ear canal sits at eye height and slightly behind, so that is two measurements instead of
two guesses. Cups widened so they sit outside the hair rather than buried in it — with
thanks for the marked-up screenshot, which showed the gap immediately.

### The dance was being driven by two staircases

`sway` came from `sin(beats * PI/2)` where `beats` was an **integer counter** — a step
function — and the beat impulse decayed a tenth per frame. Eight bones driven off those is
exactly what "jittery" looked like.

There is a continuous **phase** now, in beats, advancing at the detected tempo; each beat
only eases it a quarter of the way toward the nearest whole beat rather than driving
anything. Sway is a smooth function of phase over a bar, the per-beat bounce is a raised
cosine, and nothing anywhere is a spike. It keeps ticking when the music stops so she eases
out mid-stride instead of freezing on the last value.

### A latent bug that would have hit the first good monitor

**The canvas was laid out at the size of its drawing buffer.** `setSize(w, h, false)`
deliberately leaves the CSS alone, `#scene` relied on `inset:0`, and a canvas is a
*replaced* element — with width and height auto it takes its intrinsic size, which `inset`
cannot stretch. On any display with `devicePixelRatio > 1` the canvas therefore overflowed
its stage by exactly that ratio, anchored top-left, and she rendered enormous and off to
the bottom right.

Invisible here because WebView2 runs at dpr 1 on this machine, and waiting for the first
4K monitor or a laptop with display scaling. Found only because the browser pane emulates
dpr 2 — and confirmed pre-existing by reproducing it on the previous commit before
assuming it was today's work.

---

## 0.15.2 — 2026-08-31

**"She will not dance" is almost never the analyser.** PATCH.

Reported: music playing, no dancing, and a reasonable guess that her music sense was off by
default. It is not — `Music` defaults to true and the log showed the loopback open the
whole time. **The music was playing on a different computer**, and WASAPI loopback taps
what *this* machine plays. She was listening perfectly to an output nobody was using.

Nothing was broken, so the fix is to make the invisible thing visible:

- The Music self-test names the endpoint she actually **opened** — not
  `LoopbackListener.DefaultDevice()`, which is a different question now that `OutputDevice`
  exists — and its fix line says to check the music is coming out of that one.
- `MusicSummary` names the device even while she *is* hearing something, so the log answers
  the question before it is asked.
- The Settings hint names it too: *"Listening to Headset Earphone. Play music through that
  one — she hears a single output, not the machine."*

Verified end to end afterwards: 183 bpm at confidence 1.00 from the probe, then she wore
her headphones and danced, tracking 110 → 138 bpm as the track changed.

**Also found, and left as [[Roadmap]] stage 11a:** she cannot hear a room at all. Loopback
is what the *computer* plays; a speaker in the same room only reaches the microphone, which
is not wired to the analyser. The mic read 0.013 against a 0.004 floor, so it could hear it.
For something that lives in a room rather than on a desktop, that is the wrong way round.

*(A label/`for` double-toggle was suspected in the music checkbox and tested: one click, one
toggle, one event. Not the bug. Recorded so nobody suspects it twice.)*

---

## 0.15.1 — 2026-08-31

**The microphone was never broken.** PATCH.

`EarsTest mic` judged its reading against a single threshold, so a quiet room and a dead
device both came back `SILENT` — and it spent a morning looking like a fault. It reports
the endpoint's level and mute state now, and gives the same three verdicts the self-test
does: signal, room noise, or genuine digital silence. The Jabra reads 100%, unmuted, and a
noise floor of 0.004 that rises when spoken to. Working all along.

The same single-threshold mistake in the self-test was fixed in v0.12.0; this is the other
half of it, in the tool someone reaches for first.

---

## 0.15.0 — 2026-08-31

**The gate and the brain stop being the same model.** MINOR: a notable behaviour change,
and a documented rule reversed.

### A constraint that was the VM's, not the design's

`GateModel` had to equal `LocalModel`, and the self-test **failed** when it did not. The
reason was real and measured: on the 16 GB dev VM the server could hold one model, so a
separate gate meant a swap on every utterance — 24 seconds against 0.7 for a warm call.

Re-measured on 32 GB: **a 3B gate and a 7B brain both stay resident**, alternating calls
run at 0.39 s and 3.1 s with no swap at all, and 15.9 GB is still free. The check is a note
now rather than a failure, and the config recommends the split — the two jobs want opposite
things, and no one model was best at both.

### Bigger was not better at anything measured

New probe: `EarsTest models <name>...` times a warm gate judgement, scores it against four
mixed cases, times a spoken reply, and counts how many of four unambiguous requests produce
the right tool call. CPU-only, which is what this machine has:

| model | gate | correct | reply | tools |
|---|---|---|---|---|
| `llama3.2:3b` | 688 ms | **4/4** | 4.9 s | 4/4 |
| `qwen2.5:3b` | **308 ms** | 2/4 | **1.6 s** | 4/4 |
| `qwen2.5:7b` | 815 ms | 2/4 | 5.1 s | 4/4 |
| `llama3.1:8b` | 1843 ms | 2/4 | 9.5 s | 4/4 |

Two things worth keeping:

- **Every model called tools correctly**, including the 3B. The assumption that small
  models cannot be trusted with Stage 12 was wrong on these cases — though four
  unambiguous requests is a floor, not a hard test.
- **Both Qwens answer NO to "what is the weather doing tomorrow".** They fail closed on a
  gate that is supposed to fail open, and a gate that never opens is worse than none. It
  looks like prompt sensitivity rather than capability, and it is worth chasing:
  `qwen2.5:3b` at 308 ms would be the better gate if the instruction can be made to land.

**Configured:** gate `llama3.2:3b-cpu`, brain `qwen2.5:7b-cpu`. Gate median **665 ms** on
the split, and the brain is now a model that can hold a conversation for the same reply
time the 3B took.

### Answers folded into the roadmap

- **Google Home, no Home Assistant.** Google publishes mobile SDKs and a manufacturer API;
  there is no supported way for a Windows service to control Google Home. HA is therefore
  the recommendation *to make the stage possible*, not a preference — and most of those
  devices are Matter or WiFi underneath, which HA can hold alongside Google without
  removing anything.
- **UniFi UDM SE at `10.1.1.1`**, folded in through HA's UniFi integration rather than a
  second server.
- **The UDM SE's own WireGuard is enough; Tailscale is not needed** unless the ISP is
  CGNAT. Forwarding WireGuard's UDP port is a different proposition from forwarding hers —
  a WireGuard endpoint is silent to unauthenticated packets. The Windows firewall still
  needs one inbound rule, scoped to the LAN.

---

## 0.14.0 — 2026-08-31

**A door she can open to a phone, and a body that moves when the music does.** MINOR: a
new transport mode and a changed performance.

### Stage 13 — the groundwork, which is not the app

Steps 1–4 of the stage, all of which live in this repo. The Android client does not, and
is not pretended to.

- **`RemoteAccess`** binds every interface as well as loopback. Off by default, logged
  loudly when on, and the single riskiest line in the project — which is why it is a
  switch rather than a default.
- **`remote.key`** — a durable, readable, groupable secret in her data folder. The per-run
  token is regenerated every start, which is right for a page the host loads a second
  later and useless for a phone in a pocket. **The per-run token is not accepted from off
  the machine at all**: it is written to the log and carried in a URL. Regenerating the
  key revokes every device at once, which is the whole revocation story.
- **`subscribe`** — a face may name message types it does not want. Opt-*out*, so a face
  that never sends it keeps getting everything and no existing renderer changes. A phone
  sends `{"skip":["viseme","level"]}`, because sixty visemes a second is a battery rather
  than a feature.
- **The network decision, written down** in PROTOCOL.md: Tailscale or Wireguard, never a
  forwarded port. One shared secret in front of a microphone and a house is enough behind
  a tunnel and is not enough on the open internet.

### She dances with more than her head

Head-only movement reads as listening. A person moving to music moves from the hips, and
the shoulders and arms follow a beat late. `setDance(amount, sway, beat)` joins the avatar
interface: hips, spine, chest and arms, every bone written as rest + offset so the idle
pose is not lost the moment the music starts, and the chest counter-rotating against the
hips because a torso does that and a board does not. Small angles throughout — the line
between dancing and convulsing on an anime rig is about fifteen degrees. The bust ignores
it, having no body to move.

### Audit

- **A race in the code written an hour earlier.** The per-connection `skip` set was a
  `HashSet` written on one connection's receive thread and read by `Broadcast` from
  whichever thread produced a viseme. It is an immutable set swapped by reference now:
  replacing a reference is atomic, editing a set under a concurrent read is not.
- **`WasapiCapture` is obsolete** in this NAudio, and the peak probe was the only thing
  still using it — two capture paths in one codebase is how a diagnostic ends up
  disagreeing with the thing it diagnoses. It is `WasapiRecorderBuilder` now, the same as
  the loopback, sharing the same `AudioSamples` decoder. `Peak` became `PeakAsync`.
- The probe now says when **no buffers arrived at all**, which is the one thing a peak of
  zero cannot tell you on its own: a quiet room and a dead device read identically
  otherwise.
- Build is clean: **no errors and no warnings**.

---

## 0.13.0 — 2026-08-31

**A splash, a room with air in it, and the beginnings of hands.** MINOR: a new subsystem
and a changed console.

### Stage 11 — the interface

- **A loading splash**, held until the scene has built *and* the host has answered. She
  used to show a finished-looking console while the renderer, the socket and the voice
  were all still coming up, and the gap between looking ready and being ready is where
  every "she ignored me" report starts. It names the step it is on, shows a startup notice
  in place — a voice or a speech model downloading is the thing that actually takes the
  time — and opens anyway after fifteen seconds rather than stranding anyone behind it.
- **Typing costs a click.** The text field was the width of the window in a console where
  most turns are spoken. It is behind a keyboard button now, opens focused, closes on send
  or Escape. Hush moved out of the field, because she can be speaking while it is shut.
- **The status strip stacks bottom-left.** Five readings on one line read as a sentence to
  parse; five short lines read as a list to scan, and scanning is all that happens from a
  sofa.
- **The headphones are placed rather than guessed.** They were sized from the character's
  *height* (`headPoint.y * 0.115`) — an assumption about head width taken from an unrelated
  measurement, wrong by a different amount for every model — and added at the head bone's
  origin, which is the base of the skull, so the band hung at the neck. Both are measured
  now: the head bone against the top of the model's own bounding box.
- **The room has air.** 260 additive motes drifting in the key light, their opacity
  following the key so they vanish at night rather than reading as fog; the near parallax
  slab leans on the beat; and the camera sways a few centimetres on a very long period.
  That last one is what makes the rest work — parallax is relative motion, and against a
  bolted-down camera the layers slide while the scene still reads flat.

### Stage 12 — the tool seam

**The seam is built and tested; she cannot call a tool yet.** Both halves of that are
deliberate. See ROADMAP.md.

- `ITool` / `IToolProvider` / `ToolRegistry`, and `McpClient` speaking **MCP over stdio** —
  handshake, `tools/list`, `tools/call`, newline framing, per-request timeouts, and a read
  loop that fails pending calls when a server dies instead of hanging a turn.
- **`ToolRisk`, and the rule that dangerous tools do not run unasked.** MCP carries no risk
  annotation, so it is inferred and biased towards asking.
- `McpServers` in config; tokens belong in `Env`, not in arguments.
- `tools\mock-mcp.ps1`, a three-tool server, so the seam is testable with no house
  attached. 11 new checks drive the real client against a real child process.

The brain-side tool loop is **not** written: it changes the working conversation path and
there is no API key here to verify it against. Writing it blind and calling it done would
repeat precisely the mistake v0.12.0 spent its length undoing.

### Stage 13 — away

Designed, not built, and the design says the app is the last step rather than the first.
The prerequisites — a transport that may leave the machine, an auth secret that survives a
restart, the Tailscale-or-Wireguard decision, and a protocol subset a phone would want —
all live in this repo. The client itself is a separate project needing an Android SDK and
a device.

---

## 0.12.0 — 2026-08-31

**Four things were broken, and three of them had been blamed on something else.** MINOR:
her face, her ears' placement and her hearing of music all change behaviour.

### Her textures never loaded, and it was the CSP

`img-src` allowed `'self' data: https://octavia.avatar`, and **`blob:` was not among
them**. glTF keeps its textures inside the binary; three.js decodes them into a `Blob` and
loads them from a `blob:` URL — which that policy blocked, for every texture, in every
model, of every format. `connect-src` lacked it too, so the ImageBitmap route failed the
same way. Proven in the browser before changing anything: `IMG BLOCKED`, `FETCH BLOCKED`,
`BITMAP BLOCKED`, then all three passing after adding `blob:` to both lists.

She has 20 textured materials of 28 now, at up to 2048×2048, with zero texture errors —
against `hasMap: false` and eight failures before.

**The KTX2 theory in ROADMAP.md was wrong**, and so was the lighting explanation that
preceded it. `vrm-avatar.js` still never calls `setKTX2Loader`, which remains a real
latent gap for a model that needs it, but it was not this.

### The microphone check called a working headset silent

It read `AudioMeterInformation.MasterPeakValue`, on the reasoning that Windows' own meter
shows signal without opening a capture. **That meter reports zero unless something already
holds the device open**, so an idle machine always measured exactly 0.000. It opens a real
WASAPI capture now and reads the Jabra's true noise floor of 0.004, and reports three
states rather than two — speech, room noise, and genuine digital silence, which is the
only one that is a fault.

### The music path was decoding the wrong bytes

A crest factor of 1.7 had been read over Remote Desktop, through a virtual streaming
endpoint and through two different real sound cards, and blamed on each in turn. It was
none of them.

A shared-mode mix format is `WAVE_FORMAT_EXTENSIBLE`, not `IeeeFloat`, so the float test
failed and the decoder fell through to `ToInt16` — **taking the low two bytes of each
32-bit sample**. Those bits are uniform noise, whose RMS is 0.577 and whose crest factor
is 1.73. That is exactly what was measured, to three decimals, on every device.

Fixed by testing the sub-format GUID and decoding 8/16/24/32-bit properly, in a shared
`AudioSamples` so the diagnostics cannot drift from the capture. Against a played 132 bpm
track: crest **1.7 → 7.7**, peak 0.793 and RMS 0.103 matching the source exactly, and the
tempo **131.8 bpm at confidence 1.00** where it used to wander between 75 and 184.

`EarsTest music demo` now compares what it captured against the crest factor of the track
it played, instead of against an absolute threshold that assumed the signal was fine.

### She was thinking on a 2014 graphics card

`ollama ps` said `4%/96% CPU/GPU`: 28 of 29 layers offloaded to a **GeForce GT 730 over
Vulkan**. Ollama does not need CUDA, so being Kepler did not save it. Every attention-gate
call took ~3.9 s against an 8 s timeout, and the gate probe's median was **8009 ms** —
pegged at the timeout, failing open on all eighteen lines. She was answering the
television, and the only symptom anyone could see was that she felt slow.

The OpenAI-compatible endpoint ignores `options.num_gpu`, so placement is pinned with a
Modelfile instead — `PARAMETER num_gpu 0`, created as `llama3.2:3b-cpu`. Gate median
**8009 ms → 640 ms**; the corpus 144.2 s → 16.5 s.

- **New `Gate speed` self-test** times a warm call and names this cause, because nothing
  in the config was wrong and no amount of reading it would have found this.

### Settings that stop this happening silently again

- **`MicrophoneDevice`, `OutputDevice`, `CameraDevice`** — pick a device instead of
  inheriting the Windows default. Matched by substring, which is required: `WaveIn`
  truncates names to 31 characters, so the same headset is two different strings.
  Dropdowns in Settings, populated by the host over `hello`.
- **`WhisperCompute`** (`auto`/`cpu`/`gpu`) and **`WhisperThreads`**. Note that
  Whisper.net's own default order is CUDA-first, so "auto" is not neutral. On this
  machine CUDA never loaded at all — the GT 730 is below CUDA 12's floor — so Whisper was
  always on the CPU. Measured with `small.en`: 4 threads 5.55 s, 8 threads 4.12 s,
  16 threads 3.66 s.
- `EarsTest compute <auto|cpu|gpu> [model] [threads]` measures it rather than assuming.

### Also

- `attach-face.ps1` read the log from `%APPDATA%`, which v0.11.0 moved. It now resolves
  the data folder the same three ways `Paths.cs` does.
- `hello` reports the devices and the compute choice; `attach-face` prints them.

---

## 0.11.0 — 2026-08-30

**She keeps her things where she lives.** MINOR: every path she writes to moved.

Her data folder was `%APPDATA%\Octavia` unconditionally. On the move to the physical PC
that folder was left behind by a copy that took the repo and the vault, and its contents —
Whisper models, Piper voices, and **the only `.vrm` avatar** — were lost. The avatar was
unrecoverable: no note in the repo or the vault had ever recorded which model it was.

`Core\Paths.cs` now resolves the data folder in three steps: `OCTAVIA_DATA` if set;
otherwise `<repo>\data` whenever she is launched from a build inside the repo, found by
walking up from the executable for `Octavia.slnx`; otherwise `%APPDATA%\Octavia`. So
`dotnet run`, a Debug shortcut and `dist\Octavia.exe` all agree, copying the project now
copies her models and her face with it, and an installed copy under Program Files still
writes somewhere it is allowed to.

Everything routes through `Paths`, so this was one file and 38 call sites needed no
change — the seam was already there.

- `data\` is git-ignored and excluded from the vault snapshot: hundreds of megabytes of
  downloaded artefacts, none of it source.
- Existing data moved from `%APPDATA%\Octavia` into `<repo>\data`; the old folder is
  parked as `Octavia.old` rather than deleted.
- Two replacement avatars added — `AvatarSample_A.vrm` (VRoid, VRM 0.x) and
  `VRM1_Constraint_Twist_Sample.vrm` (pixiv, VRM 1.0). Neither uses KTX2, so neither
  reproduces the texture fault; see ROADMAP.md.
- Docs follow: `<data>` is defined once in README.md and in the vault's
  [[Profiles & Configuration]], and the literal paths elsewhere now point at it. The
  historical record in this file and the Changelog is left as it was written.

---

## 0.10.0 — 2026-08-30

**Stage 10 — the console rebuilt, and her face made legible.** MINOR: the whole control
surface changed shape.

**Partially delivered, deliberately versioned anyway.** The work was interrupted before
its follow-up items, and a snapshot whose `<Version>` said 0.9.2 while carrying an
entirely different interface would be worse than one that describes itself. ROADMAP.md
carries the full landed / outstanding split; the short version is that all six approved
design decisions are in, and the VRM texture loading, the local-first profiles and the
documentation are not.

- **Tokens.** `face.css` opens with a `:root` set — colour, type scale, spacing, radii —
  and every value in the file comes from it.
- **One drawer, four tabs** (Transcript, Settings, Health, Dev) replacing three
  hand-written drawers, each of which had its own header, close button and slide.
- **The API key lives in Settings**, not the status strip. A missing key lights an amber
  pill that opens Settings with the field focused — a guided empty state rather than a
  mystery.
- **Hush is transient**, inside the field, present only while there is something to stop.
- **The chrome sits in her light.** The day-cycle keyframes now carry the page's share of
  the hour, handed out through `onPalette` as `--room-tint`, `--room-ink` and
  `--room-line`. The window no longer floats in fixed grey above a room that moves.
- **The caption reaches distance size** (~34px from 25px), per the 10-foot-interface
  floor of about 28px at 1080p.
- **Status pills carry health dots** and the strip holds no controls.
- **"Listening Post" is now "In residence."**
- **The bust has a mouth, and eyes that are visible.** Both were buried inside the head —
  the mouth aperture closed to 1.6% of its height 0.076 behind the face surface, the iris
  reached 0.839 against a surface at 0.871. Lips are now deep forms whose front caps
  emerge and taper with the skull's curvature.
- **`setLightScale` joins the avatar interface.** A VRM is authored for roughly unit
  lighting; this room runs its key to 2.2 and clipped her to a white oval. Her material
  response is scaled instead of the room being dimmed, so the wall and the bust keep the
  light they were built for.

**Fixed:** a temporal-dead-zone error — the palette callback fires during
`createEnvironment`, before `avatar` exists — which stopped the scene building entirely.
Surfaced by `ready { faceBuilt: false }`.

## 0.9.2 — 2026-08-30

**She looks at you.** A camera button beside the microphone, wearing the same contract:
a person presses it, a marker stays visible the whole time it is on, pressing it again
ends it.

- While it is lit, her gaze follows movement — a **motion centroid over a 64×36 grid**,
  computed inside the renderer at ~8 Hz. Deliberately not a face detector: thirty lines
  a person can read, no vendored model, and it fails the way a person does — when
  nothing moves she keeps looking where she last saw you.
- **Nothing crosses the protocol.** No frame, no coordinate, no flag; the host cannot
  start it, stop it, or know it is happening. The only protocol change is `camera` in
  `hello`, so the face hides a button that could only fail.
- Two markers, two severities: the momentary red **bar** for a still, and a standing red
  **camera pill** beside the state pill for watching — the two facts a person needs at a
  glance share one corner.
- Verified live against the redirected webcam: button on → permission granted → pill up →
  head visibly turning with movement in the room; button off → device closed, pill gone.

## 0.9.1 — 2026-08-30

**The camera, against real hardware for the first time.** A webcam was redirected into the
VM, and everything 0.9.0 could only reason about became testable. Three things came out of
it, and none of them would have been found any other way.

- **The host now answers WebView2's permission requests.** There was no
  `PermissionRequested` handler at all, so the runtime decided — which made `"Camera":
  false` a suggestion rather than a boundary. The host now denies every permission except
  a camera request from her own origin with the setting on, and denies microphone,
  geolocation, notifications and the rest outright. Nothing is saved in the browser
  profile, so turning the camera off takes effect immediately.
- **`Glance` describes a captured frame without keeping it** — size, brightness and
  spread, logged on every `sight`. This is the silent microphone of Stage 4 all over
  again: a camera can open, report no error, and hand over a black rectangle, and from
  the outside that is indistinguishable from her being wrong about what she saw. A lens
  cap, a privacy shutter and an unlit room all look like success without it.
- **The capture waited two animation frames before grabbing.** Enough against a synthetic
  device, far too few against a real sensor that has auto-exposure to finish. Now 450 ms.
  Measured before and after on the same scene: brightness 0.15 → 0.18 — a real but modest
  improvement, and honestly the room is simply dark.
- The dev panel gains **Take a still**, which runs the whole camera path without needing a
  question that earns one. It is the only way to exercise the permission grant at all.

**Verified end to end:** device redirected → host granted the permission → `getUserMedia`
inside WebView2 → 768×432 frame with genuine detail (spread 0.126, well clear of the 0.02
blank threshold) → `sight` back to the host. The eyes are no longer half-verified.

## 0.9.0 — 2026-08-30

**Stage 9 — the attention gate, and eyes.**

MINOR: the gate is a new subsystem and it changes her behaviour in the most visible way
possible — she can now decline to answer.

### The gate

- **`AttentionGate`** decides whether something she overheard was addressed to her. Two
  layers, cheapest first: rules settle her name, the follow-up window and fragments for
  nothing; only ambiguous lines reach the small local model. **No paid model is ever used
  to decide whether to use a paid model.**
- **Fails open.** A companion who goes silent because a helper model died is broken; one
  who occasionally answers the television is annoying. The log says which happened.
- **Never silent.** A declined line is logged and sent to the face as `overheard` with
  the reason, and shown faintly in the transcript. "She ignored me" has to be answerable.
- **Typed input is never gated.** If you took the trouble to type it, you meant it.
- Measured over 18 labelled lines: 14 agreed, 1 ignored-you, 3 answered-noise, median
  1.2 s. `EarsTest -- gate` prints the table; `EarsTest` asserts the rules and the parser.
- Two findings that changed the design: the gate model must be the **same** model as the
  brain — a separate one is evicted and reloaded per utterance, 24 s against 0.7 s warm —
  and a **reasoning model is useless as a gate**, spending its whole budget deliberating
  and returning nothing. No portable switch turns that off.

### Eyes

- **`Situation`** replaces the loose `context` parameter on `IBrain.RespondAsync`, and
  now carries a still as well. It rides with the current question only, never the history.
- **The face owns the camera; the host owns the decision.** `look` asks for one frame,
  `sight` answers with it or with a reason. The device is released in the same breath and
  an unmissable marker shows while it is live.
- **Off by default — the only sense that is.** A microphone in a room is expected; a
  camera is not. Three cheap, auditable gates before it opens: the setting, the words, and
  whether the brain has eyes at all. None consults a model.
- `Sight.WantsEyes` is a word list rather than a classifier *on purpose*: a person must be
  able to read it and know exactly what makes her look.
- **Honestly half-verified.** The intent rules, the no-camera path, the refusal path and
  the marker are all tested. No frame has ever been captured here — this VM has no camera —
  so the picture reaching Claude is built and unproven.

### Also

- Self-test gains **Camera** and **Attention gate** checks; the latter fails loudly when
  the gate and brain models differ, because that misconfiguration is invisible and costs
  24 seconds an utterance.
- `Speech.WithoutThinking` for one-shot replies, alongside the streaming `ThinkFilter`.

**Not built:** the wake word (openWakeWord has no "Octavia", and the free layer already
matches her name for nothing), and presence detection, which needs the camera. Home
Assistant stays deferred by choice.

## 0.8.2 — 2026-08-30

**Stage 8's decision, and the contract that makes it possible.**

PATCH rather than MINOR on purpose: the stage is *decided*, not completed. Rendering waits
on a machine with a real GPU, and calling this a finished stage would misreport it.

- **`tools\attach-face.ps1 -Conformance`** drives a running host through a turn, a
  self-test and a forget, and reports which host-to-face messages arrived and whether each
  carried the fields `PROTOCOL.md` promises. Stage 8's premise — that photorealism is a
  renderer swap — was an assumption until something checked it.
- **It found a real gap on its first run.** A face attaching to a session already in
  progress was never told her current expression: `emotion` is only sent when her mood
  *changes*, and a mood can sit unchanged for many minutes. An external renderer would
  have shown the wrong face indefinitely. `hello` now carries `state`, `emotion` and
  `emotionWeight`, and the built-in face applies them on connect.
- **`PROTOCOL.md` gains "What a renderer must implement"** — what a face must handle, what
  it may ignore and what that costs, and the rates it has to survive. The checklist an
  Unreal face gets built against.
- The reconnection section now says what *is* replayed, rather than only what is not.

## 0.8.1 — 2026-08-30

**A dev panel: every performance she can give, on a button.**

Anything rare in the face — a mood, a viseme, the headphones — was awkward to look at,
because the only way to see one was to *cause* it. This is a fourth drawer that drives
`window.Face` directly, so a shape can be held still and judged.

- **State, mood, mouth, eyes, level, music, props and room**, each a row of buttons or a
  slider. `Say a line` runs a viseme sequence, because a single held shape says nothing
  about whether a mouth reads as talking.
- **`Hold the face`** stops host messages that would *move* her — `state`, `level`,
  `viseme`, `emotion`, `music` — from reaching the renderer, so a mood set by hand is not
  wiped by the next thing she says. Captions, the transcript and settings still arrive.
- **A Senses row that deliberately does leave the renderer**: listening, hush and the
  music sense are devices, and the face does not own one.
- **Offered on the `dev` profile, and whenever there is no host** — a face served on its
  own is being worked on by definition. `DevPanel` in config overrides both. The module
  is imported only when the panel is opened, so a published face never loads it.
- Three additions to the avatar-facing side of `window.Face` for things the face
  schedules for itself and could not otherwise be asked for: `blink()`, `look(x, y)` and
  `setProp('headphones', on)` — the last taking `null` to hand the prop back to the music.

## 0.8.0 — 2026-08-30

**Stage 7 — music: headphones on, dance.**

- **She hears what the machine plays.** WASAPI loopback taps the render endpoint, so she
  hears the output mix without a cable or a virtual device. This is the capability that
  stopped her being a browser page in the first place.
- **Beat detection with no model and no network.** A spectral-flux onset envelope,
  autocorrelated for a tempo, matched against a pulse train for the phase — the same
  arithmetic shape as the mouth in 0.7.0. `MusicAnalyzer` is device-free and pure, so it
  is tested against generated tracks at known tempi rather than by playing something and
  watching her.
- **Music, not speech.** Three things must agree: loud enough to be something, continuous
  enough not to be talking, and periodic at a steady rate. Speech fails the last two.
- **She keeps time while she talks.** Her own voice reaches the loopback like anything
  else, so the analysis is held while she speaks and the tempo already found keeps
  running — she stays in step with a track she is talking over, and cannot mistake
  herself for music.
- **The face responds**: headphones descend onto her head on sustained music, a nod on
  the beat, a sway across the bar, and the room's halo answers energy with a ring that
  leaves on each beat. `setHeadphones` joins the avatar interface; the bust and a VRM
  both implement it.
- **The brain is told there is music**, on the current question only — never in the
  system prompt, which would void the cache breakpoint, and never in the history, which
  would leave it claiming there is music an hour after it stopped. `IBrain.RespondAsync`
  grows an optional `context`.
- **`music` and `setMusic`** join the protocol as additive version-1 messages. No audio
  crosses it and none is kept: what survives analysis is a tempo and a loudness.
- **Settings**: a switch for whether she listens at all, off closing the device rather
  than ignoring it. The tempo appears in the status strip, because "she is not dancing"
  and "there is nothing to dance to" otherwise look identical.
- **Diagnostics**: a Music check naming the output device, and `Default output` in the
  report — "she never dances" is usually that line saying NONE.
- **`tools\serve-face.ps1`** serves `wwwroot` over loopback so the renderer can be
  developed in an ordinary browser with devtools, instead of a rebuild and a screenshot
  per change.
- Fixed on the way in: a `long.MinValue` sentinel that overflowed and stopped the tempo
  search ever running, and a beat clock that truncated fractional hops to zero so it
  never advanced while she spoke.

**Known limitation, and it is the machine's.** Remote Desktop's "Remote Audio" endpoint
normalises everything to full scale — crest factor 1.7, near square — at any volume. The
tempo cannot be found in audio with no dynamics left in it. `EarsTest -- music` measures
the crest factor and says so plainly rather than leaving it a mystery.

## 0.7.0 — 2026-08-30

**Stage 6 — a voice worth the face.**

- **`IVoice`**, alongside `IBrain` and `ISpeechRecognizer`. `VoiceBox` becomes
  `SapiVoice`; `NeuralVoice` joins it. `OctaviaSession` never learns which one it has,
  and the engine can be swapped under a running session.
- **Piper, out of process.** A long-lived child process: sentences on stdin, raw PCM on
  stdout, played through NAudio. Same reasoning as the local brain — a second ONNX
  runtime in this process would sit beside Whisper's CUDA-linked one. The 60 MB model
  loads once rather than once per sentence.
- The engine (22 MB) and the voice (~60 MB) are fetched on first use, into her data
  folder, with progress on her face. **This downloads an executable**, which is a
  different thing from downloading a model, so it happens only when the neural voice is
  asked for and the URL is in `PiperStore` where it can be read.
- **Lip sync is read out of the audio**, not from the engine. Piper reports no phoneme
  timings — and neither will most of its replacements. `VisemeReader` takes loudness for
  the jaw and the balance of three formant-ish bands for the lips. It is analysed at the
  moment each buffer reaches the sound card, so the mouth is in step with what is heard
  rather than with what has been generated.
  - Both its references adapt: loudness against a decaying peak, and the front/back axis
    against a running centre. Fixed thresholds tuned on one voice made every other voice
    mumble in a single shape.
  - `EarsTest -- mouth <file.wav>` prints the shape timeline, which is how the boundaries
    were set — a deliberately distributional choice, not a phonetic one.
- Settings → Speech chooses the engine; Settings → Voice lists that engine's voices and
  fetches one that has not been downloaded yet. `VoiceRate` maps to Piper's phoneme
  length, so speaking speed still works.
- She starts on the Windows voice and upgrades herself once the neural engine is ready,
  so a first run talks immediately instead of sitting mute through an 80 MB download.
- A small FFT in `Audio\Fft.cs`, written rather than taken from a package: thirty lines,
  no native dependency to collide with Whisper's, and Stage 7's beat detection wants the
  same thing.
- Protocol (still version 1): `setVoiceEngine` in; `hello` gained `voiceEngine`, and
  `voices[]` became `{value, label}` pairs because only the host knows how to tidy a
  Piper file name.
- Fixed: the end-of-utterance watchdog fired *during* synthesis, when the buffer was
  legitimately empty because nothing had been produced yet — she reached "idle" and then
  started talking. Nothing is over until it has begun.
- Fixed: a sound card is fed continuously and an empty buffer comes back as silence, so
  she sent a viseme twelve times a second forever, every one saying "mouth shut".
- Twelve voice checks added, and 15 face/expression checks moved to `SapiVoice`.

## 0.6.1 — 2026-08-30

**A settings menu, and the persistence bug it uncovered.**

- **Settings drawer** in the face: appearance (the bust or any `.vrm` in her avatars
  folder), voice, and the room's lighting hour. Changes apply instantly and are saved.
  Voice moved here from the console row, which now shows it as a label.
- The host lists what is actually in the avatars folder, so choosing a character is a
  dropdown rather than a filename typed into a config file. A name that is not there is
  refused rather than saved.
- `RoomHour` in config.json pins the room's lighting; negative follows the clock.
- Protocol (still version 1): `setAvatar` and `setRoomHour` in; `hello` gained
  `avatars[]`, `avatarFile` and `roomHour`.
- **Fixed: settings did not persist.** v0.4.1 stopped `Save()` flattening the profile
  overlay into the file by carrying back a *hand-kept list* of runtime-changeable
  properties — which was wrong the moment a new setting existed. `AvatarFile` and
  `RoomHour` reached the host, changed the face, logged, and were silently dropped on
  save. `Save()` now writes back every key that differs from the settings as they stood
  at load, which is the same guarantee without a list to keep in step.
- Fixed: a face that cannot reach the avatar origin retried on every `hello`, refetching
  megabytes and logging an error each time. A URL that failed is not retried.
- Fixed: switching avatars only ever loaded; it never switched back to the bust.
- Five config checks added, covering a new setting persisting, two saves in a row, and
  the overlay still not leaking.

## 0.6.0 — 2026-08-30

**Stage 5 — the real face: VRM avatar and a room to stand in.**

- **three.js r180, as ES modules.** The vendored 2021 UMD build could not host
  `@pixiv/three-vrm` (r158+), and writing the new scene against the old API would have
  meant writing it twice. `three`, `GLTFLoader`, `BufferGeometryUtils` and `three-vrm`
  are vendored under `wwwroot\lib` with their bare specifiers rewritten, so no import map
  is needed and the CSP stays `script-src 'self'`.
- **One avatar interface** — `setViseme`, `setExpression`, `setGaze`, `setBlink`,
  `setPose`, `update`. The plaster bust implements it; so does a VRM. The face owns blink
  schedules, saccades, head carriage and mood; the avatar owns how a jaw actually moves.
- **VRM characters.** Drop a `.vrm` in `%APPDATA%\Octavia\avatars` and name it in
  `AvatarFile`. The host maps that folder to a read-only `https://octavia.avatar` origin
  and offers the URL in `hello`; the face loads it once and falls back to the bust —
  loudly, into the log — if anything goes wrong. Arms are posed out of the format's
  T-pose on load, since VRM supplies a rest position rather than an idle.
- **The expression vocabulary is VRM 1.0's**, deliberately: `happy / angry / sad /
  relaxed / surprised / neutral`, and visemes `aa / ih / ou / ee / oh`. Protocol to
  character is an identity mapping with nothing in between to get wrong.
- **Visemes carry a shape.** SAPI's 21 identifiers were collapsed to one openness number;
  they now also map to a mouth shape. "aa" and "ou" are the same jaw drop with different
  mouths, and that difference is most of what makes speech look like speech.
- **`emotion` message.** Her expression is read from the text of each sentence as she
  speaks it — locally, free, no model call, per the standing rule that reflex-speed things
  stay local. The message exists so a model can override it later.
- **A room, not a backdrop.** The flat wall became a shader environment: a full-day
  lighting cycle (the wall's temperature and the key, rim and ambient lights move
  together), two drifting depth slabs for parallax, a vignette, grain, and a halo behind
  her that answers the microphone. `Face.setHour(21)` pins the clock to look at it.
- Self-test gained an **Avatar** check; "she looks wrong" becomes a filename.
- Fixed: the backdrop's vignette used `smoothstep` with its edges reversed, which is
  undefined in GLSL and rendered as no vignette at all.
- 15 face-and-expression checks added to `tools\EarsTest`, covering the viseme map's
  coverage and the mood reader's vocabulary.

## 0.5.0 — 2026-08-30

**Stage 4 — diagnostics: make her debuggable in someone else's hands.**

- **Structured logging.** `octavia.log` gains levels (`debug`/`info`/`warn`/`error`),
  rolls at 1 MB keeping three predecessors, and remembers its recent lines in memory so
  the face can show them without touching the disk. `LogLevel` in config.json;
  `OCTAVIA_LOG` writes it elsewhere.
- **Real crash handling.** The UI thread's unhandled exceptions were logged and swallowed;
  they are now logged with a stack trace *and shown on her face*. Background-thread and
  unobserved-task exceptions are caught too.
- **Self-test**, in-app and on demand: settings, transports, renderer, microphone signal,
  speech model, voice and brain. Every failing check carries the sentence that says what
  to do about it. Deliberately free — the local brain is pinged, Claude is never called.
- **"Save diagnostics"** — a file dialog and one zip: `README.txt`, `report.txt`,
  `config.json` with anything key-shaped removed, and `logs/`. Reachable from the face and
  from the tray, because the moment you most need it is when the face is what broke.
- **Privacy, stated up front.** The log contains transcripts of things said in the room.
  README.txt says so, names the file to read first, and confirms the API key is not in the
  bundle — it stays DPAPI-sealed outside it.
- **`--diagnostics <path>`** writes a bundle with no window and no session, so a machine
  where she will not start at all can still produce one. It runs before the single-instance
  check on purpose.
- A **Health panel** in the face: the checks, this machine's facts, and the recent log.
- Fixed: the save dialog was constructed on a socket thread, so it threw into a discarded
  task and did nothing whatsoever. Every fire-and-forget task now logs its own failure
  instead of waiting for the garbage collector to notice.
- Fixed: the headless bundle blocked the dispatcher thread on a task whose continuations
  were posted back to it, and hung forever.
- Fixed: the config redactor matched substrings, so it blanked `Hotkey` and `MaxTokens` —
  two of the most useful lines in a fault report. It now matches whole words of the
  setting's name and only ever redacts a *string*, since a secret is never a number.
- Protocol (still version 1): `selfTest`, `saveDiagnostics`, `openDataFolder` in;
  `diagnostics`, `diagnosticsSaved` out. A face may ask for a bundle but never names the
  path — the host owns the dialog.
- `tools\attach-face.ps1 -Send '<json>'` sends arbitrary protocol messages, and prints
  self-test results.
- 27 diagnostics checks added to `tools\EarsTest`, covering levels, rotation, the check
  set, and every guarantee the bundle makes about its own contents.

## 0.4.1 — 2026-08-30

**Profiles you can pin to a launcher.**

- `--profile <name>` (also `--profile=<name>` and `-p`) on the command line, outranking
  `OCTAVIA_PROFILE` and the `Profile` key in turn. A desktop shortcut can pass an
  argument but cannot set an environment variable, so without this a launcher had no
  way to say which rig it wanted.
- The desktop shortcut now passes `--profile dev`, so it always starts on the local
  model regardless of what the config file happens to say.
- The startup log records where the profile came from — `profile 'dev' (command line)`
  — and the tray tooltip reads `Octavia — dev (local)`.
- Fixed: saving a setting while a profile was applied wrote the *merged* values back,
  flattening the overlay into the base settings permanently. Changing her voice on the
  dev profile therefore rewrote the file's base brain to `local`, and every later run
  inherited it. Runtime changes now go back to the un-overlaid original.
- Launching a second instance while she is running ignores `--profile`; it now says so
  in the log instead of silently surfacing an Octavia on the other profile.
- `OCTAVIA_CONFIG` points her at a different settings file, so the harness can exercise
  loading and saving without touching the real one.
- Twelve config checks added to `tools\EarsTest` covering the precedence order and the
  flattening regression.

## 0.4.0 — 2026-08-30

**Stage 3 — cut the cord: the face protocol.**

- `PROTOCOL.md` — the host/face contract written down, with a `protocol` version carried
  in `hello`. Faces must ignore unknown types and fields; removing or repurposing one is
  a version bump.
- `WebSocketFaceServer` — a loopback listener (raw `TcpListener` + `WebSocket.CreateFromStream`,
  so no urlacl reservation and no elevation). Binds `127.0.0.1` only and requires a
  per-run token, compared in fixed time. Bad or missing token is refused at the handshake
  with 401 and never becomes a WebSocket.
- `FaceHub` — fans one message out to every attached face and merges what comes back.
  The session no longer knows how many renderers are listening or which transport each
  chose.
- The built-in page now prefers the socket too, so it is no longer a special case; it
  falls back to postMessage only if the port could not bind. Confirmed that a page on a
  virtual `https` origin *can* reach `ws://127.0.0.1` — loopback counts as potentially
  trustworthy, so mixed-content blocking does not apply.
- `tools\attach-face.ps1` — attach to a running Octavia as an external face and drive her,
  proving the protocol rather than the WebView2 page is the interface.
- Eight protocol checks added to `tools\EarsTest`, including both token-refusal cases and
  fan-out to two simultaneous faces.
- Fixed: the server abandoned the socket on a close frame instead of completing the
  handshake, so a face that disconnected politely saw an EOF error on its way out.
- Config: `FacePort` (default 8848; 0 picks any free port).

## 0.3.0 — 2026-08-29

**Stage 2 — a local brain, and dev profiles.**

- `IBrain` interface; the old `Brain` becomes `ClaudeBrain`, joined by `LocalBrain`.
- `LocalBrain` streams from any OpenAI-compatible server (Ollama, LM Studio,
  `llama-server`) over SSE, so swapping models is a config edit, not a rebuild.
  Kept out-of-process on purpose — a second CUDA-linked native runtime in this
  process would collide with Whisper, and later with Audio2Face.
- Shared `Conversation` and `Speech` helpers: sentence draining, a markdown
  flattener, and a streaming `<think>` filter so a reasoning model's scratchpad is
  never spoken aloud.
- Named config **profiles** merged over the base settings in memory; `OCTAVIA_PROFILE`
  overrides the file. `dev` = local brain + `small.en`; `live` = Claude +
  `large-v3-turbo`.
- Ollama installed and benchmarked on the dev VM; `llama3.2:3b` chosen as the dev
  default on wall-clock and persona adherence, not tokens/sec.
- Silence watchdog: a microphone that opens but delivers digital silence now says so
  on her face after 10 seconds instead of failing invisibly.
- `tools/EarsTest` gained 16 brain checks, a live local-brain probe, and a
  `-- mic` device diagnostic.
- Fixed: the `<think>` filter held a fixed lookahead margin, so replies shorter than
  the tag were never counted as spoken and `LocalBrain` threw "returned nothing".

## 0.2.0 — 2026-08-29

**Stage 1 — ears: VAD + Whisper.**

- `WhisperRecognizer` behind the existing `ISpeechRecognizer`: microphone →
  Silero VAD → Whisper, entirely local.
- Silero VAD (vendored ONNX) gates every utterance with pre-roll, hangover and a
  minimum voiced duration, so Whisper never sees silence and cannot hallucinate
  text out of it. Bracketed non-speech tags are filtered as a second line.
- Whisper models download once to `%APPDATA%\Octavia\models`, with progress on her
  face. CPU and CUDA runtimes both referenced; CUDA is picked up automatically.
- The Windows desktop recognizer stays as an automatic fallback.
- `tools/EarsTest` added: synthesizes speech, runs the whole pipeline headlessly,
  and asserts that silence transcribes to nothing.

## 0.1.0 — 2026-08-29

**Initial build.** Grew out of a single-file HTML prototype (`talking-avatar.html`).

- WPF / .NET 10 host with the face in a WebView2, served from a virtual
  `https://octavia.face` origin so the page is a secure context.
- Three.js plaster bust ported out of the prototype into `wwwroot`, driven entirely
  by host messages behind `IFaceTransport`.
- Claude via the Anthropic SDK, streamed and cut at sentence boundaries so she starts
  speaking before the reply is finished.
- API key sealed with DPAPI to the current Windows account — it never reaches the page.
- SAPI speech synthesis with real viseme events driving the jaw.
- Tray icon, configurable global hotkey, single-instance with window surfacing.
```
