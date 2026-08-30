---
project: Octavia
tags: [octavia, recovery]
---

# Restore From Snapshot

**Octavia has been on GitHub since 08/30/2026** — [`N3cTr0/Octavia`](https://github.com/N3cTr0/Octavia), private. **`git clone` is now the first thing to try**, and it gives you a complete tree including `wwwroot\lib`, which this snapshot deliberately does not.

The `Code\` folder in this vault is the second copy, and this note is the procedure for when it is all that survived. It remains worth keeping current: it is human-readable, it round-trips byte-for-byte on every sync, and it does not depend on GitHub or on a credential still being valid.

> **Moving to a new machine?** Read [[Moving To The New Machine]] first. If you copied `C:\Projects\Octavia` across, you do not need this note at all — it is the fallback for when the vault is all that survived, and its one real gap is `wwwroot\lib`, covered below.

## What the snapshot contains

83 notes covering every source file: C#, XAML, the `.csproj`, the app manifest, the face's HTML/CSS/JS, the scripts in `tools\`, plus `README.md`, `ROADMAP.md`, `PROTOCOL.md` and `versions.md`.

Each note carries a `source-path:` frontmatter field holding the repo-relative path, and exactly one fenced code block containing the file verbatim. That pair is all a restore needs.

**Not included, all re-downloadable:**

| Left out | Why | How to get it back |
|---|---|---|
| `wwwroot\lib\` | 3 MB of three.js and three-vrm; somebody else's code, not our source | `npm`/unpkg — the versions are named in [[The Face]] |
| `Assets\models\silero_vad.onnx` | Vendored model | Re-download; see [[Silero VAD Context Window]] |
| `data\models\` | Whisper models | Re-downloaded on first listen |
| `data\voices\` | Piper and its voices | She fetches them on first use — see [[The Voice]] |
| `data\avatars\` | VRM characters, ~10–15 MB each | **Not re-downloadable. See the warning below.** |
| `data\config.json` | Her settings, including the profiles | Recreated with defaults; copy it to keep a tuned rig |
| `bin`, `obj`, `dist` | Build output | Rebuild |

Since v0.11.0 her data lives at `<repo>\data`, so **copying the repo copies her models, voice and avatars with it** — there is no longer a second folder to remember. `data\` is git-ignored, so a `git clone` gets none of it; that is intended, but it means a clone is not a complete restore of *her*, only of her source. Leave `apikey.dat` behind either way — DPAPI makes the copy useless.

> **The avatars row is the one that has already bitten.** Every other line here re-downloads itself; a `.vrm` does not. On 08/30/2026 the only avatar was lost with the old data folder, and **no note anywhere had recorded which model it was**, so it could not be re-fetched. If you add a character, write down what it is and where it came from — in [[The Avatar Interface]] — not just the file.

**Also not in the snapshot: `Screenshots\`.** Those are a record of what was verified, not source — see [[Screenshots]].

## Restoring

For each note in `Code\` except `_Code Index.md`: read `source-path`, take the contents of the fenced block, and write it to that path under the repo root as **UTF-8 without BOM**.

```powershell
$code = 'C:\Obsidian Vaults\Vibe Projects\01 Projects\Octavia\Code'
$repo = 'C:\Projects\Octavia'
$enc  = New-Object System.Text.UTF8Encoding($false)

Get-ChildItem $code -File -Filter *.md | Where-Object Name -ne '_Code Index.md' | ForEach-Object {
  $text = [IO.File]::ReadAllText($_.FullName)
  if ($text -notmatch '(?s)source-path: (.+?)\r?\n---\r?\n') { return }
  $rel = $Matches[1].Trim()
  $m = [regex]::Match($text, '(?s)\n```[a-z]*\r?\n(.*)\r?\n```\r?\n?$')
  if (-not $m.Success) { Write-Warning "no fence: $rel"; return }

  $dest = Join-Path $repo $rel
  New-Item -ItemType Directory -Force (Split-Path $dest -Parent) | Out-Null
  [IO.File]::WriteAllText($dest, $m.Groups[1].Value, $enc)
}
```

Then:

1. Re-fetch `wwwroot\lib\` — five files, about 3 MB, and the one thing this snapshot cannot give you:

   | File | Size here |
   |---|---|
   | `three.module.js` | 589 KB |
   | `three.core.js` | 1371 KB |
   | `three-vrm.module.js` | 966 KB |
   | `GLTFLoader.js` | 112 KB |
   | `BufferGeometryUtils.js` | 35 KB |

   **Take `three.core.js` as well as `three.module.js`** — three.js splits across the two, and fetching only the first gives a 404 that surfaces as "the scene did not build". Then rewrite their bare `from 'three'` imports to relative paths, or nothing loads at all. Versions are named in [[The Face]].
2. Re-fetch `Assets\models\silero_vad.onnx` (Silero VAD).
3. `dotnet restore` — package versions are pinned in the `.csproj`.
4. `dotnet run --project tools\EarsTest` to confirm the pipeline is intact.

## Verification

The round trip is **tested, not assumed** — a lesson taken from PartnerTool, where the first snapshot browsed fine and could not restore cleanly.

The check extracts each note's fenced body exactly as the restore does and compares it byte-for-byte with the source file (both normalised to LF, source trailing whitespace trimmed the way the sync writes it), and separately scans for mojibake markers.

Last run 08/30/2026 at v0.11.0, on the new machine: **83 clean, 0 mismatched, 0 with mojibake.** (The run before it was v0.10.0 at 82, immediately before the move off the VM.)

That run earned its keep: it caught mojibake in `README.md` where an earlier `perl -pi` edit had mangled multi-byte UTF-8 down to lone `0xE2` bytes. The vault was faithfully copying a corrupted *source* file — the snapshot was fine, the repo was not. Verify after every sync that follows a bulk text edit.

## Two traps

- **Read and write with `[IO.File]::ReadAllText` / `WriteAllText`.** PS 5.1's `Get-Content`/`Set-Content` default to the ANSI code page and silently double-encode non-ASCII. If the verification uses the same lossy cmdlets as the writer, it produces self-consistent garbage and *passes*.
- **`perl -pi` without an encoding layer does the same damage**, and worse: it drops bytes rather than double-encoding, so the result is invalid UTF-8 that no mechanical reversal can recover. Prefer the editor tools or `sed` for text with box-drawing characters, dashes and arrows.

See [[Lessons Learned]] and [[Build & Release]].
