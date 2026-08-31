---
project: Octavia
tags: [octavia, handover]
---

# Overnight, 08/30–08/31/2026

*What happened while you were asleep, what needs you, and what I could not answer alone.*

She went from **v0.10.0 on a machine that would not build** to **v0.14.0**, on GitHub, with
every check passing. Four faults are fixed and three of them had been blamed on the wrong
thing. Three roadmap stages are in, one of them deliberately half-finished.

---

## Read this bit first

**Restart her.** An instance was running for most of the night holding a lock on
`Octavia.exe`, so `bin\` may still be an older build. Rebuild before judging anything:

```
dotnet build C:\Projects\Octavia\Octavia.slnx -c Debug
```

**Her config changed.** `LocalModel` and `GateModel` now point at `llama3.2:3b-cpu`
instead of `llama3.2:3b`. That single change is worth **8009 ms → 640 ms** per utterance.
The old file is at `data\config.json.bak`.

**Her data folder moved** into the repo at `C:\Projects\Octavia\data`. The old one is
parked at `%APPDATA%\Octavia.old` (125 MB) — delete it when you are happy.

---

## Getting her onto this machine

| | |
|---|---|
| **Memory** | Was keyed to the old home path and stale — thought she was at Stage 4. Rebuilt and corrected. |
| **NuGet** | First build failed completely, `NU1100` on every package. The user-level config has an empty `<packageSources>`; added a repo `nuget.config`, same fix PartnerTool already had. |
| **Toolchain** | .NET SDK 10.0.400, Ollama 0.33.2, `gh` 2.98.0 installed. PowerShell 7.6.5 and WebView2 were already present. |
| **Git** | `git init`, then a **private GitHub repo**. That was the "one real risk" the handover note flagged — and it was well aimed. |
| **Mark of the Web** | 1107 files in the repo carried it; cleared. |
| **Execution policy** | Blocks the repo's unsigned scripts here. Use `pwsh -ExecutionPolicy Bypass -File`. The machine policy was left alone. |
| **Shortcut** | Recreated. Note your Desktop is OneDrive-redirected. |

---

## The four faults, and what they actually were

### Her textures never loaded — it was the page's own CSP

`img-src` allowed `'self' data: https://octavia.avatar` and **not `blob:`**; neither did
`connect-src`. glTF keeps textures inside the binary, three.js loads them from a `blob:`
URL, so every texture in every model of every format was blocked. She had **never once**
rendered with a face.

Proven in the browser before changing anything, and again after. **20 of 28 materials
textured, zero errors.**

The KTX2 theory was wrong. So was the lighting theory before it.

### The microphone check called a working headset silent

It read the endpoint's peak meter, which **reports zero unless something already holds the
device open**. An idle machine always measured exactly `0.000`. It opens a real capture
now and reports three states — speech, room noise, digital silence — because only the last
is a fault.

### The music path was decoding the wrong bytes

A crest factor of 1.7 had been blamed on Remote Desktop, then a virtual streaming
endpoint, then a headset. It was none of them.

A mix format is `WAVE_FORMAT_EXTENSIBLE`, not `IeeeFloat`, so the float test failed and the
decoder read **the low two bytes of each 32-bit sample**. Those bits are uniform noise —
whose RMS is 0.577 and crest is 1.73, which is what was measured to three decimals on
*four unrelated devices*.

Crest **1.7 → 7.7**. Tempo **131.8 against a played 132 at confidence 1.00**, where it used
to wander between 75 and 184.

### She was thinking on a 2014 graphics card

`ollama ps` read `4%/96% CPU/GPU` — **28 of 29 layers on the GT 730 over Vulkan**. Ollama
does not need CUDA, so being Kepler did not save it. Every gate call took ~3.9 s against an
8 s timeout, so **the gate failed open on all eighteen test lines**: she was answering the
television and the only symptom was feeling slow.

Gate median **8009 ms → 640 ms**. Your instinct about the CPU was exactly right.

---

## Stages 11, 12, 13

**Stage 11.** A loading splash that names the step it is waiting on and opens anyway after
fifteen seconds. Typing behind a button. Status stacked bottom-left. Headphones measured
against the head rather than guessed from character height. Motes in the key light and a
slow camera sway — the sway is what makes the existing parallax layers read as depth.

**Stage 12 — the seam is built and tested; she cannot call a tool yet.** `McpClient` over
stdio, a risk policy that refuses dangerous tools unconfirmed, `mock-mcp.ps1` so it is
testable with no house attached, 11 checks against a real child process. **The brain-side
tool loop is not written** — it changes her working conversation path and there is no API
key here to verify it against.

**Stage 13 — the groundwork, not the app.** `RemoteAccess` binds every interface;
`remote.key` is durable where the per-run token is not, and the per-run token is **not
accepted from off the machine at all**; `subscribe` lets a phone decline visemes. The
Android client is a separate project needing an SDK and a device.

**She dances with her body now** — hips, spine, chest and arms, counter-rotating, every
bone as rest + offset so the idle pose survives.

---

## The audit

- **A race in code I had written an hour earlier**: the per-connection `skip` set was a
  `HashSet` mutated on one thread and read by `Broadcast` on another. Now an immutable set
  swapped by reference.
- `WasapiCapture` is obsolete; the peak probe was the last user. Moved to the same builder
  and the same decoder as the loopback.
- `attach-face.ps1` still read the log from `%APPDATA%` after the data move — a regression
  from my own change.
- **Build is clean: no errors, no warnings.** Full suite passes.

---

## What needs you

1. **Paste the API key** into Settings. She is on the local brain until then, and the
   Claude path is the one thing I could not test all night.
2. **Delete `%APPDATA%\Octavia.old`** when you are satisfied.
3. **Is the Jabra's microphone muted?** The probe reads `0.0001` — buffers arrive, so the
   capture works, but there is essentially no signal. That is either a silent room at 2am
   or a mute switch. Run `dotnet run --project tools\EarsTest -- mic` and talk.
4. **Re-take the Stage 10 exit test from the sofa**, now that she has a face worth looking
   at. Still outstanding from the move.

## Answered, 08/31/2026 morning

- **`%APPDATA%\Octavia.old` deleted.**
- **The key is the Anthropic one** (`sk-ant-…`), Settings → API key.
- **A newer graphics card by the end of the year.** So the room's effects stay cheap for
  now and Stage 8 stays parked, but neither has to be designed around the GT 730 forever.
  Re-run `EarsTest models` and `EarsTest compute` the day it arrives — every figure in this
  vault is CPU-only.
- **The avatar stays a borrowed sample** until there is time to buy one.
- **UniFi is a UDM SE at `10.1.1.1`.**
- **No Home Assistant; smart devices are on Google Home.** This is the one answer that
  changed a plan — see [[Roadmap]] stage 12. Google has no API a Windows service can use,
  so HA becomes a prerequisite rather than a preference.
- **WireGuard on the UDM SE is enough**; Tailscale only if the ISP is CGNAT.
- **The model question is now measured** rather than argued — gate and brain are split,
  and bigger turned out to be worse at everything tested. See [[Profiles & Configuration]].

## Still open

- **Is the Jabra's microphone muted?** The probe reads 0.0001 with buffers arriving, so the
  capture works and there is no signal. Run `dotnet run --project tools\EarsTest -- mic`
  and talk.
- **Go-ahead on the brain-side tool loop.** The key answers *which* key, not *yes write it*.
- **The Stage 10 exit test from the sofa**, still outstanding from the move.

## Questions I could not answer alone

1. **The tool loop — shall I write it against your key?** It is the last thing between the
   seam and her actually running the house. I stopped because it changes the conversation
   path and I could not verify it.
2. **Which local model for the house?** `llama3.2:3b` is fine for chat and thin for tool
   calling. A 7–8B would be better at picking the right tool and slower per turn — and this
   GPU cannot help, so it is all CPU. Worth benchmarking before Stage 12 goes further.
3. **Tailscale or Wireguard?** Step 3 of Stage 13, and everything about the phone follows
   from it. I have written down *that* it must be one of them, not which.
4. **Home Assistant — do you already run it?** If so, which version: the built-in MCP
   server arrived in 2025.x and would mean writing no integration code at all.
5. **UniFi — controller or UDM, and on what address?** I deliberately did **not** scan your
   network to find out.
6. **The avatar.** She is a borrowed pixiv sample. Your stated end-goal is a photoreal
   woman with headphones; VRoid Studio is free and would at least make her *yours* in the
   meantime. Worth an hour?
7. **The GT 730.** It cannot do Stage 8, it is slower than the CPU for the model, and it is
   currently the reason the room's effects have to stay cheap. A modern card changes three
   stages at once. Is that on the cards, or should I keep optimising for this one?

## Not done, and why

- **The brain-side tool loop** — needs a key.
- **The Android app** — needs an SDK and a device, and belongs in its own repository.
- **UniFi and Home Assistant servers** — blocked on questions 4 and 5, and on the loop.
- **Screenshots for v0.13/v0.14** — only the texture fix is captured. The dance and the
  splash are verified but not photographed for the vault.
