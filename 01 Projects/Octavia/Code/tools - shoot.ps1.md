---
project: Octavia
tags: [octavia, code]
source-path: tools\shoot.ps1
---

# tools\shoot.ps1

```powershell
<#
.SYNOPSIS
  Capture Octavia's window to the vault's Screenshots folder, named for the current version.

.DESCRIPTION
  The visual record of the project. Most releases are code, but the ones that are not are
  exactly the ones nobody can reconstruct later from a diff - and a year of changelog
  entries about the chrome is no substitute for being able to see her.

  Every existing shot is 1100x780, so this sizes the window to match before capturing.
  A set taken at a consistent size can be flipped through; one taken at whatever size the
  window happened to be cannot.

  The version is read from Directory.Build.props - one number for the whole of her since
  Stage 15 split her across three projects - so the file name cannot drift from the release.

  She must already be running - this does not launch her, because what is worth
  photographing is usually a state you have just driven her into by hand.

.PARAMETER Caption
  What the shot shows, in the same voice as the existing ones: 'the drawer on Settings',
  'idle with the placard faded'. Becomes 'v0.19.3 - <caption>.png'.

.PARAMETER Delay
  Seconds to wait after the window comes forward. Give her a moment when the shot needs
  an animation to finish or the placard to fade.

.PARAMETER VaultPath
  Root of the Obsidian vault. Defaults to $env:OCTAVIA_VAULT, else
  'C:\Obsidian Vaults\Vibe Projects'.

.PARAMETER KeepSize
  Capture the window at whatever size it already is, rather than resizing to 1100x780.
  For the rare shot that is about the window being a different shape.

.EXAMPLE
  pwsh -File tools\shoot.ps1 'idle, the placard faded' -Delay 11
#>
[CmdletBinding()]
param(
  [Parameter(Mandatory, Position = 0)][string]$Caption,
  [int]$Delay = 2,
  [string]$VaultPath = $(if ($env:OCTAVIA_VAULT) { $env:OCTAVIA_VAULT } else { 'C:\Obsidian Vaults\Vibe Projects' }),
  [switch]$KeepSize,
  [switch]$Settings
)

# The settings window has its own shape and 1100x780 would stretch it into a window that
# never existed, so -Settings implies -KeepSize unless the caller overrode the size on
# purpose. Her face is the thing that has to stay comparable shot to shot.
if ($Settings) { $KeepSize = $true }

$ErrorActionPreference = 'Stop'

Add-Type -AssemblyName System.Drawing
Add-Type -AssemblyName System.Windows.Forms

# SetForegroundWindow fails silently when called from a background process, and the
# capture then photographs whatever window *is* in front. AttachThreadInput first, and
# verify afterwards - this has cost a debugging round before. See Lessons Learned.
Add-Type @'
using System;
using System.Runtime.InteropServices;
public static class Win {
  [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);
  [DllImport("user32.dll")] public static extern IntPtr GetForegroundWindow();
  [DllImport("user32.dll")] public static extern uint GetWindowThreadProcessId(IntPtr h, IntPtr pid);
  [DllImport("user32.dll")] public static extern bool AttachThreadInput(uint a, uint b, bool attach);
  [DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr h, int cmd);
  [DllImport("user32.dll")] public static extern bool MoveWindow(IntPtr h, int x, int y, int w, int t, bool repaint);
  [DllImport("user32.dll")] public static extern bool GetWindowRect(IntPtr h, out RECT r);
  [DllImport("kernel32.dll")] public static extern uint GetCurrentThreadId();
  [StructLayout(LayoutKind.Sequential)] public struct RECT { public int Left, Top, Right, Bottom; }
}
'@

$repo = Split-Path $PSScriptRoot -Parent

# The version comes from the project file, never from a parameter: a screenshot filed
# under the wrong release is worse than no screenshot.
$csproj  = Join-Path $repo 'Directory.Build.props'
$version = ([xml](Get-Content -LiteralPath $csproj -Raw)).Project.PropertyGroup.Version |
           Where-Object { $_ } | Select-Object -First 1

if (-not $version) { throw "no <Version> in $csproj" }

$shots = Join-Path $VaultPath '01 Projects\Octavia\Screenshots'
if (-not (Test-Path -LiteralPath $shots)) { throw "no Screenshots folder at '$shots'" }

# Her face first, and her server's settings window second. The settings window is a real
# part of the visual record - two releases running, it was the only thing that changed -
# and both times it got photographed by hand because this only knew about the client.
$names = if ($Settings) { 'Octavia.Server', 'Octavia' } else { 'Octavia', 'Octavia.Server' }

$octavia = $names | ForEach-Object { Get-Process $_ -ErrorAction SilentlyContinue } |
           Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1

if (-not $octavia) {
  Write-Host 'shoot: no Octavia window is open.'
  Write-Host '       her face:  start the client. her settings: Octavia.Server.exe --settings'
  Write-Host '       a tray with no window open has no MainWindowHandle, which looks the same from here.'
  exit 1
}

Write-Host "shoot: photographing $($octavia.ProcessName)"

$handle = $octavia.MainWindowHandle

[void][Win]::ShowWindow($handle, 9)   # SW_RESTORE, in case she is minimised

$mine  = [Win]::GetCurrentThreadId()
$mainw = [Win]::GetWindowThreadProcessId($handle, [IntPtr]::Zero)

[void][Win]::AttachThreadInput($mine, $mainw, $true)
[void][Win]::SetForegroundWindow($handle)
[void][Win]::AttachThreadInput($mine, $mainw, $false)

Start-Sleep -Milliseconds 400

if ([Win]::GetForegroundWindow() -ne $handle) {
  Write-Host 'shoot: her window would not come forward; refusing to photograph whatever is in front of it.'
  exit 1
}

if (-not $KeepSize) {
  $r = New-Object Win+RECT
  [void][Win]::GetWindowRect($handle, [ref]$r)
  [void][Win]::MoveWindow($handle, $r.Left, $r.Top, 1100, 780, $true)
  Start-Sleep -Milliseconds 600   # the renderer re-frames on resize; let it settle
}

if ($Delay -gt 0) { Start-Sleep -Seconds $Delay }

$rect = New-Object Win+RECT
[void][Win]::GetWindowRect($handle, [ref]$rect)

$width  = $rect.Right - $rect.Left
$height = $rect.Bottom - $rect.Top

$bitmap   = New-Object System.Drawing.Bitmap $width, $height
$graphics = [System.Drawing.Graphics]::FromImage($bitmap)
$graphics.CopyFromScreen($rect.Left, $rect.Top, 0, 0, $bitmap.Size)

# Windows forbids \ / : * ? " < > | in a file name, and the captions here use colons freely.
$safe = ($Caption -replace '[\\/:*?"<>|]', '-').Trim()
$path = Join-Path $shots "v$version - $safe.png"

$bitmap.Save($path, [System.Drawing.Imaging.ImageFormat]::Png)
$graphics.Dispose()
$bitmap.Dispose()

Write-Host "shoot: v$version - $safe.png  ($width x $height)"
```
