---
project: Octavia
tags: [octavia, recovery, migration]
---

# Moving To The New Machine

*Written 08/30/2026, at v0.10.0, on the way off the VM.* Read this first.

She has only ever run on a **VMware VM reached over Remote Desktop**, and a surprising amount of what is written about her behaviour is really written about *that*. Moving to a physical PC does not just relocate her — it removes three documented limitations and re-opens one whole roadmap stage. The second half of this note is about that.

## Copy these three folders

The vault alone is enough to rebuild, but it is the slow path. Copying all three means nothing is downloaded twice and nothing has to be reassembled.

| Copy | From | Why |
|---|---|---|
| **The repo** | `C:\Projects\Octavia` | Everything, including `wwwroot\lib` — which the vault deliberately excludes |
| **The vault** | `C:\Obsidian Vaults\Vibe Projects` | Same path on the new machine. The notes, and the fallback snapshot |
| **Her data** | `%APPDATA%\Octavia` | ~710 MB of Whisper models, Piper voices and the avatar. Saves a long first run |

> **Superseded 08/30/2026 — there are now two folders, not three.** Her data moved into
> `<repo>\data`, so copying the repo copies her models, voice and avatars with it. The
> third row applies only to a machine still on the old layout. See
> [[Profiles & Configuration]] for the resolution rule. This change was made *because* of
> what went wrong on this move: her data folder lived somewhere the repo copy did not
> reach, and its contents were lost.

`bin`, `obj` and `dist` inside the repo are build output — skip them if you are being tidy, keep them if it is easier. They are rebuilt by `dotnet build`.

**Leave `apikey.dat` behind.** It is DPAPI-sealed to one Windows account on one machine, so the copy would be an undecryptable blob. She handles this properly: the read fails, it is logged, and she asks for a key as though new. Paste it in again — Settings → API key. See [[Conventions & Security Model]].

## What she needs installed

| Needs | Version here | Notes |
|---|---|---|
| .NET SDK | **10.0.400** | `dotnet --version`. The `.csproj` targets `net10.0-windows10.0.19041.0` |
| WebView2 runtime | — | Present on Windows 11 by default. She names it in a message if missing |
| PowerShell 7 | **7.6.5** | For the vault scripts and any text work. **Use the plain ZIP release**, not the Store/MSIX one — see [[Lessons Learned]] |
| Ollama | **0.33.2** | `ollama serve`, then `ollama pull llama3.2:3b` |

The Piper speech engine and the Whisper models download themselves on first use if you did not copy `%APPDATA%\Octavia`.

## If only the vault made it

Follow [[Restore From Snapshot]]. It reproduces every source file byte-for-byte and is round-trip tested on every sync. The one thing it cannot give you is `wwwroot\lib` — five vendored files, about 3 MB — which have to be re-fetched and have their bare `from 'three'` imports rewritten to relative paths. That step has a trap in it that already cost a debugging round, and it is written down there.

## Three limitations that were the VM, not her

This is the part worth reading carefully, because several notes in this vault describe behaviour that should simply **change** on real hardware.

### Music should start working properly

Remote Desktop's *Remote Audio* endpoint normalises everything to full scale — measured **crest factor 1.7**, essentially a square wave — at any playback volume. A beat cannot be found in audio with no dynamics left in it, so her tempo detection wandered badly while locking to within 0.3 bpm offline. See [[Music]].

On a real sound card this should simply resolve. **Re-run it early:**

```
dotnet run --project tools\EarsTest -- music demo
```

Expect the crest factor to rise well above 2.5 and the limiter warning to stop appearing. If the tempo now settles near the played 132 bpm, the honest caveat in [[Music]] and the roadmap can be narrowed to "over Remote Desktop" instead of standing as a general limitation.

### The microphone should stop being a problem

The RDP capture path needed *Local Resources → Remote audio → Record from this computer*, set before connecting, and the self-test names that setting by its full menu path. On a physical machine with a real microphone that whole failure mode goes away — though the check itself stays, because it is a good check.

### The camera changes character

Here it was a redirected webcam over RDP, and the *Browser pane blocks camera access by policy*, so the only place capture could ever be tested was inside WebView2. On the new machine an ordinary local camera should work in both. Worth re-confirming:

- one still — Dev tab → **Take a still**, then look for the `sight:` line in the log
- watching — the eye button beside the microphone

See [[Eyes]].

## The stage this re-opens

**[[The Photoreal Decision]] is blocked only on hardware.** The decision itself is made and written down: MetaHuman in Unreal for rendering, Audio2Face-3D for animation, and Audio2Face running **in the host, out of process**, because the host already has the PCM in hand at the `MouthTap`.

If the new machine has an NVIDIA card, check it against the shopping list at the end of that note: **CUDA 12.8–12.9, TensorRT 10.13+, 4 GB+ free** after Whisper and the local gate have taken theirs, and **Unreal Engine 5.7 or later**.

One dated item: the **MetaHuman Creator web app shuts down 11/05/2026**, after which character creation is in-editor only. That is a few weeks out from this note.

Two other things a real GPU changes immediately:

- **Whisper CUDA becomes real.** The `large-v3-turbo` model is the conversation default and was never practical on three cores.
- **The attention gate gets much faster.** It measured a 1.2 s median here; most of that is prompt processing on a weak CPU. See [[The Attention Gate]].

## Settings worth revisiting once you are there

- **`WhisperModel`** — the `dev` profile pins `small.en` because this VM could not do better. A real machine wants `large-v3-turbo`.
- **`GateModel`** must stay equal to `LocalModel`. Two models cannot both be resident and the server swaps them on every utterance — measured at 24 s against 0.7 s warm. The self-test fails loudly if they differ.
- **`Camera`** is currently `true` in the live config, set while testing. It ships `false` by default; decide which you want.
- **The desktop shortcut** at `C:\Users\Claude\Desktop\Octavia.lnk` points at the Debug exe with `--profile dev`. Recreate it — see [[Build & Release]] — and note the profile names are due to change; the roadmap's Stage 9 section explains why.

## The one real risk

**There is still no git repository.** The vault snapshot has been the only off-machine copy of the source for this whole project, and a machine move is exactly the moment that matters. It has been deliberately deferred, and the deferral has held up fine — but on the new machine, `git init` and a first commit costs a minute and removes the single largest structural risk here.

## What actually happened on arrival — 08/30/2026

The repo and the vault both copied across intact: `check-vault.ps1` reported **VAULT OK,
82/82 notes restoring byte-for-byte**, which is as good a verification of the copy as
exists. The new box is `N3CTR0-PC`, account `N3cTr0`, **16 logical cores and 32 GB** against
the VM's two or three. Project and vault paths are unchanged; only `%APPDATA%` moved.

**Installed:** .NET SDK **10.0.400** and Ollama **0.33.2** with `llama3.2:3b`, both via
winget. PowerShell **7.6.5** and the WebView2 runtime were already present — and PowerShell
is machine-wide at `C:\Program Files\PowerShell\7`, so the MSIX/AppData-virtualisation trap
that bit us on the VM does not apply here. Claude Code also runs **elevated** here, which
reverses the per-user-installs-only constraint the old dev-VM note describes.

**Two things this note did not predict:**

1. **The first build failed completely** — `NU1100` for every package, including the Windows
   SDK reference pack. The user-level NuGet config on this machine has an empty
   `<packageSources>` and Octavia had no repo-level config, so there was no source to
   restore from. Fixed by adding `nuget.config`, the same fix PartnerTool already carries.
   `sync-vault.ps1` did not collect `*.config`, so that file would have been absent from
   Restore From Snapshot — the one file that makes a fresh machine build, missing from
   exactly the scenario it exists for. Both are fixed and the snapshot is now 83 notes.
2. **The execution policy blocks the unsigned repo scripts here.** `check-vault.ps1` and
   `sync-vault.ps1` fail with `SecurityError: not digitally signed`. Run them as
   `pwsh -NoProfile -ExecutionPolicy Bypass -File <script>` — a per-process flag. The
   machine-wide policy was deliberately left alone.

**Verified working:** `dotnet build` succeeds (one pre-existing `CA2024` warning in
`LocalBrain.cs`), `EarsTest` reports **ALL CHECKS PASSED**, and the local brain answers —
*"Hello, I'm Octavia, nice to meet you."*

**The one real risk is closed.** `git init` is done, identity set to match the vault repo,
and there are two commits: v0.10.0 exactly as it arrived, then the NuGet fix. Local only —
no remote yet.

### Still outstanding after the move

- **`%APPDATA%\Octavia` did not come across.** Whisper models and Piper voices re-download
  themselves, but **the `.vrm` avatar does not** — and it is the subject of the top
  outstanding bug, so the texture work is blocked until it is re-supplied. She falls back
  to the plaster bust meanwhile, which is the shipped default.
- **The API key needs pasting into Settings** — expected, and it is why `apikey.dat` was
  left behind.
- The old VM is on **`H:` → `\\10.1.1.40\c`**, but a mapped drive does not cross the
  elevation boundary, so an elevated Claude session cannot see it and the share prompts for
  credentials. Copy from an ordinary non-elevated session instead.

### Two of the three "it was the VM" hopes need revising

- **Music was re-run, and the crest factor is still exactly 1.7** — the same number the VM
  gave. That looks at first like the diagnosis was wrong, but it is not: the probe reports
  `default output: Speakers (Steam Streaming Speakers)`, and that is a **virtual endpoint**
  from the Steam/Sunshine streaming stack doing precisely what Remote Audio did. The real
  lesson is broader than the note claimed — **it was never Remote Desktop specifically, it
  is any virtual render endpoint**, and this machine boots with one as default. There is a
  Realtek card and a Jabra EVOLVE 20 MS headset attached; set one of those as the default
  playback device and re-run `dotnet run --project tools\EarsTest -- music demo` before
  drawing any conclusion about beat detection. Tempo still wandered here (75–184 bpm around
  a played 132), which is the expected consequence of a crest factor that low.
- **The camera cannot be re-verified: there is no webcam attached to this machine.** The
  Jabra headset does mean there is a real microphone, so the mic path can be tested.
- **The photoreal stage is still blocked, and the shopping list is not met.** The GPU is a
  **GeForce GT 730** — Kepler, on the legacy 30.0.14.7514 driver, with `nvidia-smi` failing
  at `Failed to initialize NVML`. CUDA 12 dropped Kepler, so **Whisper CUDA, TensorRT 10.13
  and Unreal 5.7 are all out of reach on this card**. The genuine win here is the 16 CPU
  cores, not the GPU. [[The Photoreal Decision]] stays blocked on hardware, and the
  roadmap's "if the new machine has an NVIDIA card" check should be read as answered *no*.

## Where the work stands

v0.10.0 is a **partial Stage 10**: all six approved design decisions are built, the console is rebuilt, and the bust's face is fixed. Three things are outstanding and they are listed in [[Roadmap]] under "Still to do" — the VRM's textures failing to load (which is the real reason her features are faint, not the lighting), the local-first profile change, and the documentation catch-up.

Everything builds and `ALL CHECKS PASSED` as of this snapshot.
