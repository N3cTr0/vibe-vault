---
project: Octavia
tags: [octavia, code]
source-path: tools\check-vault.ps1
---

# tools\check-vault.ps1

```powershell
<#
.SYNOPSIS
  Report whether the vault's Octavia snapshot is current, and verify it restores cleanly.

.DESCRIPTION
  Two checks, both cheap enough to run after every change set:

    Staleness - is any source file newer than the vault's _Code Index?
    Round trip - does every Code note still reproduce its source byte-for-byte,
                 and is any note carrying mojibake?

  The round trip still matters with a git remote in place: GitHub is the primary
  off-machine copy, and the vault snapshot is the readable one that `Restore From
  Snapshot` is written against.

  Exit code is the number of problems found, so it can gate a commit.

.PARAMETER VaultPath
  Root of the Obsidian vault. Defaults to $env:OCTAVIA_VAULT, else
  'C:\Obsidian Vaults\Vibe Projects'.

.EXAMPLE
  pwsh -File tools\check-vault.ps1
#>
[CmdletBinding()]
param(
  [string]$VaultPath = $(if ($env:OCTAVIA_VAULT) { $env:OCTAVIA_VAULT } else { 'C:\Obsidian Vaults\Vibe Projects' })
)

$ErrorActionPreference = 'Stop'

$repo  = Split-Path $PSScriptRoot -Parent
$code  = Join-Path $VaultPath '01 Projects\Octavia\Code'
$index = Join-Path $code '_Code Index.md'

if (-not (Test-Path -LiteralPath $index)) {
  Write-Host "check-vault: no snapshot at '$code' - run tools\sync-vault.ps1 first."
  exit 1
}

$problems = 0

# --- staleness ---------------------------------------------------------------
$sources = Get-ChildItem $repo -Recurse -File -Include *.cs,*.xaml,*.csproj,*.manifest,*.js,*.css,*.html,*.ps1,*.md |
  Where-Object {
    $_.FullName -notmatch '\\obj\\' -and
    $_.FullName -notmatch '\\bin\\' -and
    $_.FullName -notmatch '\\dist\\' -and
    $_.Name -ne 'three.min.js'
  }

$newest  = $sources | Sort-Object LastWriteTime -Descending | Select-Object -First 1
$snapped = (Get-Item $index).LastWriteTime

'newest source : {0}  ({1:MM/dd/yyyy HH:mm:ss})' -f $newest.Name, $newest.LastWriteTime
'vault synced  : {0:MM/dd/yyyy HH:mm:ss}' -f $snapped

if ($snapped -lt $newest.LastWriteTime) {
  Write-Host 'STALE - run tools\sync-vault.ps1'
  $problems++
} else {
  Write-Host 'current'
}

# --- round trip --------------------------------------------------------------
$notes = Get-ChildItem $code -File -Filter *.md | Where-Object Name -ne '_Code Index.md'
$clean = 0

# Built from char codes rather than written literally, so this script's own snapshot
# does not match the pattern and report itself as corrupt.
$mojibake = '{0}|{1}{2}' -f [char]0xC2, [char]0xE2, [char]0x20AC

foreach ($n in $notes) {
  $text = [IO.File]::ReadAllText($n.FullName)

  # 5.1's cmdlets would report these falsely; [IO.File] reads real UTF-8.
  if ($text -match $mojibake) { Write-Host "  mojibake: $($n.Name)"; $problems++ }

  if ($text -notmatch '(?s)source-path: (.+?)\r?\n---\r?\n') {
    Write-Host "  no source-path: $($n.Name)"; $problems++; continue
  }
  $rel = $Matches[1].Trim()
  $src = Join-Path $repo $rel
  if (-not (Test-Path -LiteralPath $src)) {
    Write-Host "  source gone: $rel (run sync-vault.ps1 -PruneOrphans)"; $problems++; continue
  }

  $m = [regex]::Match($text, '(?s)\n```[a-z]*\r?\n(.*)\r?\n```\r?\n?$')
  if (-not $m.Success) { Write-Host "  no fence: $($n.Name)"; $problems++; continue }

  $restored = $m.Groups[1].Value -replace "`r`n", "`n"
  $original = ([IO.File]::ReadAllText($src).TrimEnd()) -replace "`r`n", "`n"
  if ($restored -ceq $original) { $clean++ }
  else { Write-Host "  differs: $rel"; $problems++ }
}

"round trip    : $clean/$($notes.Count) notes restore clean"

# --- the visual record -------------------------------------------------------
# Most releases change no pixels, and those do not need a shot. But a release that
# changed her appearance and was never photographed cannot be reconstructed later from
# a diff, and the changelog describing it is no substitute for seeing her. So this
# reports rather than fails: it is a reminder to look, not a rule about every version.
$csproj  = Join-Path $repo 'Directory.Build.props'
$version = ([xml](Get-Content -LiteralPath $csproj -Raw)).Project.PropertyGroup.Version |
           Where-Object { $_ } | Select-Object -First 1

$shots = Join-Path $VaultPath '01 Projects\Octavia\Screenshots'

if ($version -and (Test-Path -LiteralPath $shots)) {
  $forThis = @(Get-ChildItem -LiteralPath $shots -Filter "v$version - *.png" -ErrorAction SilentlyContinue)

  if ($forThis.Count -gt 0) {
    "screenshots   : $($forThis.Count) for v$version"
  }
  else {
    $latest = Get-ChildItem -LiteralPath $shots -Filter '*.png' -ErrorAction SilentlyContinue |
              Sort-Object LastWriteTime | Select-Object -Last 1
    $since = if ($latest) { ($latest.BaseName -split ' - ')[0] } else { 'never' }
    "screenshots   : none for v$version (newest is $since) - pwsh -File tools\shoot.ps1 '<what it shows>'"
  }
}

Write-Host ''
Write-Host $(if ($problems -eq 0) { 'VAULT OK' } else { "$problems PROBLEM(S)" })
exit $problems
```
