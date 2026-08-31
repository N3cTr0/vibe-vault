---
project: Octavia
tags: [octavia, code]
source-path: tools\poke.ps1
---

# tools\poke.ps1

```powershell
<#
.SYNOPSIS
  Click a point inside Octavia's window, in window coordinates.

.DESCRIPTION
  For driving her into a state worth photographing - the drawer open, the typing field
  down - without a human at the keyboard. Coordinates are relative to the top-left of
  her window, which is what you measure off a shot from tools\shoot.ps1, so the two
  scripts agree by construction.

  Only useful beside shoot.ps1. Anything a person would do more than twice belongs in
  the protocol or the dev panel instead; this exists so the screenshot set can be
  retaken in one command after the chrome moves.

.PARAMETER X
  Pixels from the left edge of her window.

.PARAMETER Y
  Pixels from the top edge of her window.

.PARAMETER Type
  Text to type after the click, for a field that was just opened.

.PARAMETER Scroll
  Wheel notches at that point instead of clicking - negative scrolls down. For reaching
  a setting below the fold without clicking a control on the way past.

.EXAMPLE
  pwsh -File tools\poke.ps1 1036 67

.EXAMPLE
  pwsh -File tools\poke.ps1 880 400 -Scroll -5
#>
[CmdletBinding()]
param(
  [Parameter(Mandatory, Position = 0)][int]$X,
  [Parameter(Mandatory, Position = 1)][int]$Y,
  [string]$Type,
  [int]$Scroll = 0
)

$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName System.Windows.Forms

Add-Type @'
using System;
using System.Runtime.InteropServices;
public static class Poke {
  [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);
  [DllImport("user32.dll")] public static extern IntPtr GetForegroundWindow();
  [DllImport("user32.dll")] public static extern uint GetWindowThreadProcessId(IntPtr h, IntPtr pid);
  [DllImport("user32.dll")] public static extern bool AttachThreadInput(uint a, uint b, bool attach);
  [DllImport("user32.dll")] public static extern bool GetWindowRect(IntPtr h, out RECT r);
  [DllImport("user32.dll")] public static extern bool SetCursorPos(int x, int y);
  [DllImport("user32.dll")] public static extern void mouse_event(uint f, int x, int y, int d, IntPtr e);
  [DllImport("kernel32.dll")] public static extern uint GetCurrentThreadId();
  [StructLayout(LayoutKind.Sequential)] public struct RECT { public int Left, Top, Right, Bottom; }
}
'@

$octavia = Get-Process Octavia -ErrorAction SilentlyContinue |
           Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1

if (-not $octavia) { Write-Host 'poke: Octavia is not running.'; exit 1 }

$handle = $octavia.MainWindowHandle

# Same lesson as shoot.ps1: a click that lands on the wrong window is worse than no
# click, because it looks like the app ignored it.
$mine  = [Poke]::GetCurrentThreadId()
$hers  = [Poke]::GetWindowThreadProcessId($handle, [IntPtr]::Zero)
[void][Poke]::AttachThreadInput($mine, $hers, $true)
[void][Poke]::SetForegroundWindow($handle)
[void][Poke]::AttachThreadInput($mine, $hers, $false)
Start-Sleep -Milliseconds 350

if ([Poke]::GetForegroundWindow() -ne $handle) {
  Write-Host 'poke: her window would not come forward; refusing to click into whatever is in front of it.'
  exit 1
}

$r = New-Object Poke+RECT
[void][Poke]::GetWindowRect($handle, [ref]$r)

[void][Poke]::SetCursorPos(($r.Left + $X), ($r.Top + $Y))
Start-Sleep -Milliseconds 120

if ($Scroll -ne 0) {
  # Wheel rather than click: reaching a control below the fold must not press whatever
  # happens to be under the cursor on the way there.
  for ($i = 0; $i -lt [Math]::Abs($Scroll); $i++) {
    [Poke]::mouse_event(0x0800, 0, 0, (120 * [Math]::Sign($Scroll)), [IntPtr]::Zero)
    Start-Sleep -Milliseconds 60
  }
  Write-Host "poke: scrolled $Scroll at ($X, $Y) in her window"
  exit 0
}

[Poke]::mouse_event(0x0002, 0, 0, 0, [IntPtr]::Zero)   # left down
[Poke]::mouse_event(0x0004, 0, 0, 0, [IntPtr]::Zero)   # left up

if ($Type) {
  Start-Sleep -Milliseconds 400
  [System.Windows.Forms.SendKeys]::SendWait($Type)
}

Write-Host "poke: clicked ($X, $Y) in her window$(if ($Type) { " and typed '$Type'" })"
```
