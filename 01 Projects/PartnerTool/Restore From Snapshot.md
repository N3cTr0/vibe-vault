---
project: PartnerTool
tags: [partnertool, disaster-recovery]
---

# Restore From Snapshot (disaster recovery)

> **Since 2026-07-18 the code lives on git/GitHub (`N3cTr0/PartnerTool`, private).** The primary
> recovery path is now `git clone https://github.com/N3cTr0/PartnerTool.git` — see [[New Machine Bootstrap]].
> This vault snapshot is the **offline fallback** for when GitHub isn't reachable, and doubles as the
> in-Obsidian searchable source browser.

If the repo is ever lost, the vault can rebuild it. **This procedure was round-trip-tested 18 Jul 2026 at v0.19.14** — every source file restored content-identical to the live repo (diff clean, line-endings normalized). The snapshot now regenerates **automatically on every commit** (see *Regenerating the snapshot* below); `_Code Index` shows the live file count.

## What the vault holds

- `Code\*.md` — every source file (`source-path:` frontmatter + fenced code block): all `.cs`, `.xaml`, `.csproj`, `app.manifest`, `Product.wxs`, the HUS reference script, `versions.md`.
- `Code\_Assets\` — the **binary** resources (`logo.png`, `logo.ico`) the csproj needs.
- [[Changelog]] — current `versions.md` copy.

**Not** captured (the sync globs `.cs/.xaml/.csproj/.wxs/.manifest/.ps1` + `versions.md`): the build/publish + repo scaffolding — `publish.bat`, `publish-singlefile.bat`, `Properties\PublishProfiles\FolderInstall.pubxml`, `.gitattributes`, `.gitignore`, and the `.githooks\` hook. Git/GitHub is the primary recovery path and has all of them, and the exact publish commands are in [[Build & Release]] — widen the sync glob (`tools\sync-vault.ps1`) if you want the offline fallback byte-complete. (`_Code Index` shows the current snapshot file count.)

## Restore script (PowerShell 5.1)

**Use .NET file IO, not `Get-Content`/`Set-Content`** — in PS 5.1 those default to the ANSI code
page and silently mangle every non-ASCII character (·, —, ▶, ● …) in a UTF-8 file. `[IO.File]`
read/write handles UTF-8 correctly.

```powershell
$code = "C:\Obsidian Vaults\Vibe Projects\01 Projects\PartnerTool\Code"
$out  = "C:\Users\graemel\Projects\PartnerTool"   # rebuild target
$enc  = New-Object System.Text.UTF8Encoding($false)   # UTF-8, no BOM

foreach ($note in Get-ChildItem $code -Filter *.md | Where-Object Name -ne "_Code Index.md") {
  $lines = ([IO.File]::ReadAllText($note.FullName) -replace "`r`n","`n").Split("`n")
  $srcLine = $lines | Where-Object { $_ -like "source-path: *" } | Select-Object -First 1
  if (-not $srcLine) { continue }
  $rel = $srcLine.Substring("source-path: ".Length)
  $fenceIdx = @(); for ($i=0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match '^``{2,}') { $fenceIdx += $i } }
  if ($fenceIdx.Count -lt 2) { continue }
  $body = $lines[($fenceIdx[0]+1)..($fenceIdx[-1]-1)] -join "`r`n"
  $dest = Join-Path $out $rel
  New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
  [IO.File]::WriteAllText($dest, $body, $enc)
}
# Binary assets
New-Item -ItemType Directory -Force "$out\PartnerTool\Resources" | Out-Null
Copy-Item "$code\_Assets\*" "$out\PartnerTool\Resources\" -Force
```

Then build per [[Build & Release]] (`dotnet build` / single-file `dotnet publish`).

## Regenerating the snapshot (repo → vault)

**Automated (2026-07-18).** `tools\sync-vault.ps1` in the repo rewrites every `Code\*.md`, refreshes `_Code Index.md`, and mirrors `versions.md` → [[Changelog]]. A **post-commit hook** (`.githooks\post-commit`, enabled with `git config core.hooksPath .githooks`) runs it after every commit, so the snapshot never drifts. It no-ops on machines without the vault; point it at a moved vault with `-VaultPath` or `$env:PARTNERTOOL_VAULT`. Run by hand any time (e.g. after editing outside a commit) with `powershell -File tools\sync-vault.ps1` (add `-PruneOrphans` to delete notes whose source is gone).

The portable script **supersedes the reference snippet below** (which hardcodes the old host path), but the logic is identical — one note per source file, orphans flagged:

```powershell
$repo = "C:\Users\graemel\Projects\PartnerTool"
$code = "C:\Obsidian Vaults\Vibe Projects\01 Projects\PartnerTool\Code"
$enc  = New-Object System.Text.UTF8Encoding($false)
function Lang($e){switch($e){'.cs'{'csharp'}'.xaml'{'xml'}'.csproj'{'xml'}'.wxs'{'xml'}'.manifest'{'xml'}'.md'{'markdown'}'.ps1'{'powershell'}default{''}}}
$files = Get-ChildItem $repo -Recurse -File -Include *.cs,*.xaml,*.csproj,*.wxs,*.manifest,*.ps1 |
    Where-Object { $_.FullName -notmatch '\\obj\\' -and $_.FullName -notmatch '\\bin\\' }
$files += Get-Item "$repo\versions.md"
$expected = @{}
foreach ($f in $files) {
  $rel = $f.FullName.Substring($repo.Length + 1)
  $note = ($rel -replace '\\',' - ') + '.md'; $expected[$note] = $true
  $src = [IO.File]::ReadAllText($f.FullName)
  $body = "---`nproject: PartnerTool`ntags: [partnertool, code]`nsource-path: $rel`n---`n`n# $rel`n`n" +
          '```' + (Lang $f.Extension) + "`n" + $src.TrimEnd() + "`n" + '```' + "`n"
  [IO.File]::WriteAllText((Join-Path $code $note), $body, $enc)
}
# orphans = notes whose source is gone (review, then delete)
Get-ChildItem $code -File -Filter *.md | Where-Object { $_.Name -ne '_Code Index.md' -and -not $expected.ContainsKey($_.Name) } | ForEach-Object { "ORPHAN: $($_.Name)" }
```

Note the naming rule: the note file is the repo-relative path with `\` → ` - ` plus `.md`
(`PartnerTool\Pages\NetworkPage.xaml.cs` → `PartnerTool - Pages - NetworkPage.xaml.cs.md`). The restore
script below reverses it via each note's `source-path:` frontmatter. Also regenerate `_Code Index.md`
and refresh [[Changelog]].

## Keeping the snapshot fresh

Regenerate `Code\` + refresh [[Changelog]] whenever meaningful code changes land. Two gotchas, both learned the hard way (see [[Lessons Learned]]):
1. **Read/write with `[IO.File]::ReadAllText` / `::WriteAllText`, never `Get-Content`/`Set-Content`.** PS 5.1's cmdlets default to the ANSI code page and double-encode UTF-8 — the snapshot *looked* fine and even round-trip-tested green (self-consistent mojibake), but a real restore would produce garbled source. The round-trip check must also use `[IO.File]::ReadAllText` on both sides, or it hides the bug.
2. **Precompute `$open = $fence + $lang`** — in PS 5.1 array literals the comma binds tighter than `+`, which once split the fence and language onto separate lines and silently broke restores.

Round-trip-test after regenerating: restore to a temp dir and diff against the repo (with `[IO.File]::ReadAllText`).

## What is NOT in the vault

- `dist\` build outputs and `obj/bin` (rebuildable).
- `.obsidian` app config, `settings.json` runtime file (regenerated with defaults).
