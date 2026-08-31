---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: tools\check-vault.ps1
---

# tools\check-vault.ps1

```powershell
<#
.SYNOPSIS
  Report whether the vault's Octavia Android snapshot is current, and verify it restores
  cleanly.

.DESCRIPTION
  The sibling of `tools\check-vault.ps1` in her repo, with the same contract:

    Staleness  - is any source file newer than the vault's _Code Index?
    Round trip - does every Code note still reproduce its source byte-for-byte,
                 and is any note carrying mojibake?
    Screenshots - is there a shot for the current version? Reported, never enforced.

  Exit code is the number of problems found, so it can gate a commit.

  Run it after sync-vault.ps1, in the same change set as the code. A snapshot that is only
  refreshed when someone remembers is a snapshot nobody can trust.

.PARAMETER VaultPath
  Root of the Obsidian vault. Defaults to $env:OCTAVIA_VAULT, else
  'C:\Obsidian Vaults\Vibe Projects'.

.EXAMPLE
  pwsh -NoProfile -ExecutionPolicy Bypass -File tools\check-vault.ps1
#>
[CmdletBinding()]
param(
  [string]$VaultPath = $(if ($env:OCTAVIA_VAULT) { $env:OCTAVIA_VAULT } else { 'C:\Obsidian Vaults\Vibe Projects' })
)

$ErrorActionPreference = 'Stop'

$repo  = Split-Path $PSScriptRoot -Parent
$proj  = Join-Path $VaultPath '01 Projects\Octavia Android'
$code  = Join-Path $proj 'Code'
$index = Join-Path $code '_Code Index.md'

if (-not (Test-Path -LiteralPath $index)) {
  Write-Host "check-vault: no snapshot at '$code' - run tools\sync-vault.ps1 first."
  exit 1
}

$problems = 0

# --- staleness ---------------------------------------------------------------
$sources = Get-ChildItem $repo -Recurse -File -Include *.kt,*.kts,*.xml,*.toml,*.pro,*.ps1,*.md |
  Where-Object {
    $_.FullName -notmatch '\\build\\' -and
    $_.FullName -notmatch '\\\.gradle\\' -and
    $_.FullName -notmatch '\\\.idea\\'
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

# Built from char codes rather than written literally, so this script's own snapshot does
# not match the pattern and report itself as corrupt.
$mojibake = '{0}|{1}{2}' -f [char]0xC2, [char]0xE2, [char]0x20AC

foreach ($n in $notes) {
  $text = [IO.File]::ReadAllText($n.FullName)

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
# An app's appearance is not reconstructable from a diff either. Reported, not enforced.
$versionsText = [IO.File]::ReadAllText((Join-Path $repo 'versions.md'))
$version = [regex]::Match($versionsText, '(?m)^##\s+(\d+\.\d+\.\d+)').Groups[1].Value
$shots = Join-Path $proj 'Screenshots'

if ($version -and (Test-Path -LiteralPath $shots)) {
  $forThis = @(Get-ChildItem -LiteralPath $shots -Filter "v$version - *.png" -ErrorAction SilentlyContinue)

  if ($forThis.Count -gt 0) {
    "screenshots   : $($forThis.Count) for v$version"
  }
  else {
    $latest = Get-ChildItem -LiteralPath $shots -Filter '*.png' -ErrorAction SilentlyContinue |
              Sort-Object LastWriteTime | Select-Object -Last 1
    $since = if ($latest) { ($latest.BaseName -split ' - ')[0] } else { 'never' }
    "screenshots   : none for v$version (newest is $since) - adb exec-out screencap -p > <shot>.png"
  }
}

Write-Host ''
Write-Host $(if ($problems -eq 0) { 'VAULT OK' } else { "$problems PROBLEM(S)" })
exit $problems
```
