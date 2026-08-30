---
project: Octavia
tags: [octavia, conventions]
---

# Conventions & Security Model

## Standing constraints

These are load-bearing. Breaking one is a design smell, not a shortcut.

1. **Anything reflex-speed is local.** Lip sync, level meters, and (later) dancing never call a model. The API bill should track *conversation*, not *liveliness*.
2. **The key never reaches a renderer.** New capabilities land in the host.
3. **Renderer changes leave `OctaviaSession` untouched; sense changes leave the face untouched.**
4. **Native runtimes that link CUDA stay out-of-process where practical.** One process should not host two of them — see [[Choosing a Local Model]].

## Security model

- **The API key** is DPAPI-sealed (`DataProtectionScope.CurrentUser`) to one Windows account on one machine, at `<data>\apikey.dat`. `ANTHROPIC_API_KEY` takes precedence when set. It is written *in* from the face and never sent back out — the `hello` message carries a boolean, never the value.
- **It does not travel.** Copying `apikey.dat` to another machine yields an undecryptable blob. This is intended. She degrades gracefully: the read fails, is logged, and she asks for a key as though new.
- **The face is sandboxed by CSP**: `default-src 'none'; script-src 'self'; style-src 'self'` — and `connect-src` names only the loopback face socket and the read-only `https://octavia.avatar` origin the host maps to her avatars folder. It cannot reach the wider network even if something in it wanted to, and it cannot read a character file the host did not offer it.
- **A face may ask for a diagnostics bundle but never says where it goes.** The host raises its own file dialog. A face that could name the destination would be a face that could write the log — transcripts and all — anywhere on the machine.
- **Downloading an executable is treated differently from downloading a model.** The neural speech engine is fetched only when the neural voice is asked for, into her data folder rather than anywhere on the PATH, from a URL written plainly in `PiperStore.cs` — see [[The Voice]].
- **The diagnostics bundle is the one thing that leaves the building**, so its contents are a decided list rather than whatever was lying around, and it says on its front page that the log holds transcripts of things said in the room. See [[Diagnostics]].
- **The page is a secure context** via `SetVirtualHostNameToFolderMapping` to `https://octavia.face`, not `file://` — required for camera and microphone permissions in later stages.
- **The camera is off by default, and it is the only sense that is.** A microphone in a room is expected; a camera is not. It opens only when three cheap, *readable* conditions hold — the setting, the words, and whether the brain has eyes — and none of them consults a model, because the decision to open a camera in someone's home must be auditable by reading it. One frame, device released in the same breath, an unmissable marker while live, and nothing stored. See [[Eyes]].
- **Nothing acoustic leaves the machine.** Whisper and Silero both run locally. This is a deliberate improvement on the prototype, whose Web Speech API streamed the microphone to Google.
- **The loopback hears everything the machine plays**, which could be a call as easily as a song. So the analysis is local and keeps nothing: buffers are measured and dropped, and what survives is a tempo and a loudness. **No message in the protocol carries audio**, and the diagnostics bundle contains none. The setting that turns it off *closes the device* rather than ignoring it — a switch that left the capture running would be a worse promise than no switch at all. See [[Music]].

## Versioning

Pre-release `0.x`. **PATCH by default** (`0.x.y`) for fixes, tweaks and copy changes. MINOR (`0.x.0`) only for a new subsystem or notable behaviour change — a completed roadmap stage qualifies by definition; "I added a thing" does not. Bump `src\Octavia.App\Octavia.App.csproj` `<Version>` and `versions.md` together.

## Code and docs

- **American English** in all documentation.
- **Comments stay minimal.** Rationale belongs in the commit message and in this vault, not scattered through the source. Comment the *non-obvious why* — the 64-sample VAD context, the out-of-process decision — never the what.
- **Displayed dates are `MM/DD/YYYY`.** `versions.md` changelog headers use ISO `YYYY-MM-DD`, an internal doc convention only.
- **PowerShell: use `pwsh` (7.6) for interactive and diagnostic work.** It defaults to UTF-8. Windows PowerShell 5.1 reads UTF-8 as ANSI, so `Get-Content` on any vault or repo note reports mojibake that is not there — it produced a false corruption alarm on `Changelog.md` on 08/29/2026. Verify text with `pwsh`, `grep`, or `[IO.File]::ReadAllText`, never 5.1's cmdlets.
- **Scripts that may run from a git hook stay 5.1-compatible** — `tools\sync-vault.ps1` is deliberately so, since a target machine may not have pwsh. That is safe because it uses `[IO.File]::ReadAllText` / `WriteAllText` rather than the cmdlets. Portability, not an encoding compromise.
- **Never hand-edit anything under `Code\`** in this vault — `tools\sync-vault.ps1` overwrites it.

## Vault upkeep

The auto-synced half (`Code\`, `_Code Index`, `Changelog`) creates a false impression that the vault is current. **The hand-written half rots silently.** When a change set adds or changes a feature, update the relevant `Features/` note and add any expensive lesson to [[Lessons Learned]] *in the same change set* — not as an occasional catch-up.

Run `tools\sync-vault.ps1 -PruneOrphans` after a source file is renamed or removed; the plain run reports orphans but never deletes them. It caught `VoiceBox.cs` after the rename to `SapiVoice.cs`.

**Screenshots go in `Screenshots\` named `v<version> - <subject>.png`, and the caption says what was being *checked*** — not what is on screen. A picture with no question attached is a gallery; a picture with a question attached is evidence. See [[Screenshots]].

**Three notes are generated and must not be hand-edited:** [[Changelog]], [[Roadmap]] and [[Face Protocol]] are rewritten from `versions.md`, `ROADMAP.md` and `PROTOCOL.md` on every sync. They were hand-copied until v0.6.0 and drifted every single time the repo changed.

## Testing

Every subsystem gets a headless harness before it is trusted. `tools\EarsTest` proves the VAD → Whisper pipeline against synthesized speech, asserts silence transcribes to nothing, exercises the streaming text filters, checks config-profile precedence *and persistence*, the face protocol, every guarantee the diagnostics bundle makes about its own contents, the viseme and mood maps, and the audio-derived mouth — then probes the local brain. Each section found a real bug on its first run.

Two of them are **probes rather than assertions**, because some things can only be judged by looking:

| Probe | Answers |
|---|---|
| `EarsTest -- mic` | Which capture devices exist, which one Windows considers default, and whether Windows itself sees any signal |
| `EarsTest -- mouth <wav>` | The lip-sync shape timeline for a piece of speech, and the distribution the thresholds were set from |
| `EarsTest -- music` | What she makes of what is playing, and — the part that matters — how completely and how *faithfully* the audio path delivered it |
| `attach-face.ps1 -Conformance` | Whether the host actually says everything a renderer must hear, with the fields the protocol promises. Run it after changing the host, and before writing a new face |

And two rules learned the expensive way.

**Test that a setting persists, not that it applies.** The settings menu looked entirely correct — dropdown, log line, face changing — while saving nothing at all.

**Measure the input before doubting the algorithm.** When something that works offline fails against a real device, the device is a suspect. A capture diagnostic should report signal *quality* beside frame count: "we received everything" and "what we received is usable" are different claims, and only the second one was false. See [[Music]].
