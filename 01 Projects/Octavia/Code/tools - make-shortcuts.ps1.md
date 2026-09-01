---
project: Octavia
tags: [octavia, code]
source-path: tools\make-shortcuts.ps1
---

# tools\make-shortcuts.ps1

```powershell
<#
.SYNOPSIS
  Create or refresh Octavia's desktop shortcuts.

.DESCRIPTION
  Since v0.26.0 she is two executables - a server that is her, and a client that looks at
  her - so there are two icons and the order matters: the server has to be up before the
  client has anything to attach to.

  Written as a script rather than done by hand because the paths here are easy to get
  wrong twice: the Desktop is OneDrive-redirected on this machine, so
  'C:\Users\<you>\Desktop' does not exist, and the client's *assembly* is called
  Octavia.exe while its project is Octavia.App. Both have cost time before.

  Re-run it after moving the repo, and after any change to which profile the icons should
  name. It overwrites, so running it twice is safe.

.PARAMETER ProfileName
  Which rig the server shortcut pins. Named on the shortcut rather than left to
  config.json, because she rewrites that file herself - so an unnamed icon's brain can
  change without anyone touching the icon. See 'Profiles & Configuration' in the vault.

  Not called -Profile: $PROFILE is an automatic variable in PowerShell, and shadowing it
  inside a script is a trap for whoever edits this next.

.PARAMETER Dist
  Point the shortcuts at dist\ instead of the Debug build. The default is Debug on
  purpose: it is rewritten by every build, so the icons are always the latest Octavia and
  can never go stale. dist is for handing to someone else.

.PARAMETER Minimised
  Start the server's console window minimised. Off by default - the first thing anybody
  does with a new server is watch it start, and the window is also how you stop it.

.EXAMPLE
  pwsh -NoProfile -ExecutionPolicy Bypass -File tools\make-shortcuts.ps1
.EXAMPLE
  pwsh -NoProfile -ExecutionPolicy Bypass -File tools\make-shortcuts.ps1 -ProfileName cloud
#>
[CmdletBinding()]
param(
  [string]$ProfileName = 'home',
  [switch]$Dist,
  [switch]$Minimised,
  [string]$Desktop = [Environment]::GetFolderPath('Desktop')
)

$ErrorActionPreference = 'Stop'

$repo = Split-Path -Parent $PSScriptRoot
$tfm  = 'net10.0-windows10.0.19041.0'

if ($Dist) {
  $serverExe = Join-Path $repo 'dist\Octavia.Server.exe'
  $clientExe = Join-Path $repo 'dist\Octavia.exe'
} else {
  $serverExe = Join-Path $repo "src\Octavia.Server\bin\Debug\$tfm\Octavia.Server.exe"
  $clientExe = Join-Path $repo "src\Octavia.App\bin\Debug\$tfm\Octavia.exe"
}

foreach ($exe in @($serverExe, $clientExe)) {
  if (-not (Test-Path -LiteralPath $exe)) {
    throw "not built: $exe`n  run: dotnet build Octavia.slnx"
  }
}

if (-not (Test-Path -LiteralPath $Desktop)) { throw "no Desktop at '$Desktop'" }

$shell = New-Object -ComObject WScript.Shell

function Set-Shortcut {
  param($Name, $Target, $Arguments, $Description, $Window, $Icon)

  $path = Join-Path $Desktop "$Name.lnk"
  $link = $shell.CreateShortcut($path)

  $link.TargetPath       = $Target
  $link.Arguments        = $Arguments
  $link.WorkingDirectory = Split-Path -Parent $Target
  $link.Description      = $Description
  $link.WindowStyle      = $Window
  if ($Icon) { $link.IconLocation = $Icon }
  $link.Save()

  Write-Output ("shortcut: {0,-22} -> {1} {2}" -f "$Name.lnk", (Split-Path -Leaf $Target), $Arguments)
}

# 1 = normal, 7 = minimised. The client is a window and always wants 1.
$serverWindow = if ($Minimised) { 7 } else { 1 }

Set-Shortcut -Name 'Octavia Server' -Target $serverExe -Arguments "--profile $ProfileName" `
  -Description "Octavia's server - start this first, then Octavia" `
  -Window $serverWindow -Icon "$serverExe,0"

# No --profile: a profile is a brain, a Whisper model and a set of tool servers, and the
# client has none of those. It ignored the argument silently after the v0.26.0 split, which
# is exactly the kind of stale shortcut this script exists to stop.
Set-Shortcut -Name 'Octavia' -Target $clientExe -Arguments '' `
  -Description 'Octavia - attaches to her server' `
  -Window 1 -Icon "$clientExe,0"

Write-Output ''
Write-Output "  server rig : $ProfileName"
Write-Output "  build      : $(if ($Dist) { 'dist' } else { 'Debug' })"
Write-Output "  desktop    : $Desktop"
Write-Output ''
Write-Output '  Start the server first. The client says so on screen if it cannot find one.'
```
