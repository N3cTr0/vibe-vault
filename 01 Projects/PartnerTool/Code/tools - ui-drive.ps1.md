---
project: PartnerTool
tags: [partnertool, code]
source-path: tools\ui-drive.ps1
---

# tools\ui-drive.ps1

```powershell
param(
    [string]$Page = "",          # nav item name to click, e.g. "Updates"; empty = no navigation
    [string]$Shot = "shot.png",  # screenshot filename (saved next to this script)
    [int]$SettleSeconds = 3      # wait after navigation before the screenshot
)
$ErrorActionPreference = "Stop"
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes, System.Drawing
Add-Type -Namespace Native -Name Dpi -MemberDefinition '[DllImport("user32.dll")] public static extern bool SetProcessDPIAware();'
[Native.Dpi]::SetProcessDPIAware() | Out-Null

$proc = Get-Process PartnerTool* | Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1
if (-not $proc) { throw "PartnerTool window not found" }

$root = [System.Windows.Automation.AutomationElement]::RootElement
$cond = New-Object System.Windows.Automation.PropertyCondition(
    [System.Windows.Automation.AutomationElement]::ProcessIdProperty, $proc.Id)
$win = $root.FindFirst([System.Windows.Automation.TreeScope]::Children, $cond)
if (-not $win) { throw "UIA window not found" }

# Foreground it so the screenshot isn't occluded
Add-Type -Namespace Native -Name Win -MemberDefinition '[DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);'
[Native.Win]::SetForegroundWindow([IntPtr]$proc.MainWindowHandle) | Out-Null
Start-Sleep -Milliseconds 400

if ($Page -ne "") {
    $nameCond = New-Object System.Windows.Automation.PropertyCondition(
        [System.Windows.Automation.AutomationElement]::NameProperty, $Page)
    $item = $win.FindFirst([System.Windows.Automation.TreeScope]::Descendants, $nameCond)
    if (-not $item) { throw "nav item '$Page' not found" }
    $pattern = $null
    if ($item.TryGetCurrentPattern([System.Windows.Automation.SelectionItemPattern]::Pattern, [ref]$pattern)) {
        $pattern.Select()
    } elseif ($item.TryGetCurrentPattern([System.Windows.Automation.InvokePattern]::Pattern, [ref]$pattern)) {
        $pattern.Invoke()
    } else {
        # click its center as a fallback
        $r = $item.Current.BoundingRectangle
        Add-Type -Namespace Native -Name Mouse -MemberDefinition @'
[DllImport("user32.dll")] public static extern bool SetCursorPos(int x, int y);
[DllImport("user32.dll")] public static extern void mouse_event(uint f, uint dx, uint dy, uint d, UIntPtr e);
'@
        [Native.Mouse]::SetCursorPos([int]($r.X + $r.Width/2), [int]($r.Y + $r.Height/2)) | Out-Null
        [Native.Mouse]::mouse_event(2, 0, 0, 0, [UIntPtr]::Zero)  # down
        [Native.Mouse]::mouse_event(4, 0, 0, 0, [UIntPtr]::Zero)  # up
    }
    Start-Sleep -Seconds $SettleSeconds
}

$rect = $win.Current.BoundingRectangle
$bmp = New-Object System.Drawing.Bitmap([int]$rect.Width, [int]$rect.Height)
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen([int]$rect.X, [int]$rect.Y, 0, 0, $bmp.Size)
$out = Join-Path $PSScriptRoot $Shot
$bmp.Save($out, [System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $bmp.Dispose()
Write-Output "saved $out ($([int]$rect.Width)x$([int]$rect.Height))"
```
