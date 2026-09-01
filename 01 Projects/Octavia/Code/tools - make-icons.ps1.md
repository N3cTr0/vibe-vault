---
project: Octavia
tags: [octavia, code]
source-path: tools\make-icons.ps1
---

# tools\make-icons.ps1

```powershell
<#
  Cuts the icon panels out of the mock sheet, masks the black surround away, and writes
  multi-resolution .ico files.

  The mask is a *shape*, not a colour key. Keying black to transparent would punch holes
  straight through the artwork, which is mostly dark; clipping to the rounded square or
  the circle the design already has removes the surround and leaves the art untouched.

  ICO entries are PNG-compressed at every size. Windows has supported that since Vista and
  this app targets Windows 10/11 only, so the alternative — hand-rolling DIB entries with
  their AND masks — would be work done to support machines she cannot run on.
#>
param(
  [string]$Sheet = 'C:\shared\Octavia Mock Logos.png',
  # Host-side: the .ico files Windows wants for the exe and the tray.
  [string]$OutDir = 'C:\Projects\Octavia\src\Octavia.App\Assets',
  # Face-side: the icons a browser wants, which have to be reachable at `/` — since
  # v0.20.0 the socket serves the page, so a phone shortcutting to her gets these.
  [string]$WebDir = 'C:\Projects\Octavia\src\Octavia.Core\wwwroot'
)

$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName System.Drawing

New-Item -ItemType Directory -Force -Path $OutDir | Out-Null
New-Item -ItemType Directory -Force -Path $WebDir | Out-Null

function Get-Panel {
  param($Sheet, [int]$X, [int]$Y, [int]$W, [int]$H)
  $src = [System.Drawing.Image]::FromFile($Sheet)
  $out = New-Object System.Drawing.Bitmap $W, $H, ([System.Drawing.Imaging.PixelFormat]::Format32bppArgb)
  $g = [System.Drawing.Graphics]::FromImage($out)
  $g.DrawImage($src, (New-Object System.Drawing.Rectangle 0,0,$W,$H),
               (New-Object System.Drawing.Rectangle $X,$Y,$W,$H), [System.Drawing.GraphicsUnit]::Pixel)
  $g.Dispose(); $src.Dispose()
  return $out
}

# Renders the panel into a square of `size`, clipped to its own silhouette.
function Render {
  param($Panel, [int]$Size, [string]$Shape)

  $bmp = New-Object System.Drawing.Bitmap $Size, $Size, ([System.Drawing.Imaging.PixelFormat]::Format32bppArgb)
  $g = [System.Drawing.Graphics]::FromImage($bmp)
  $g.SmoothingMode = [System.Drawing.Drawing2D.SmoothingMode]::AntiAlias
  $g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
  $g.PixelOffsetMode = [System.Drawing.Drawing2D.PixelOffsetMode]::HighQuality

  $path = New-Object System.Drawing.Drawing2D.GraphicsPath
  if ($Shape -eq 'circle') {
    $path.AddEllipse(0, 0, $Size, $Size)
  }
  else {
    # Matches the corner radius the artwork already draws, so the mask lands on the
    # existing edge rather than cutting a second one just inside it.
    $r = [Math]::Max(2, [int]($Size * 0.22))
    $d = $r * 2
    $path.AddArc(0, 0, $d, $d, 180, 90)
    $path.AddArc($Size - $d, 0, $d, $d, 270, 90)
    $path.AddArc($Size - $d, $Size - $d, $d, $d, 0, 90)
    $path.AddArc(0, $Size - $d, $d, $d, 90, 90)
    $path.CloseFigure()
  }

  $g.SetClip($path)
  $g.DrawImage($Panel, 0, 0, $Size, $Size)
  $g.Dispose(); $path.Dispose()
  return $bmp
}

function Write-Ico {
  param($Panel, [int[]]$Sizes, [string]$Shape, [string]$Path)

  $pngs = foreach ($s in $Sizes) {
    $bmp = Render -Panel $Panel -Size $s -Shape $Shape
    $ms = New-Object System.IO.MemoryStream
    $bmp.Save($ms, [System.Drawing.Imaging.ImageFormat]::Png)
    $bmp.Dispose()
    ,@{ Size = $s; Bytes = $ms.ToArray() }
  }

  $fs = [System.IO.File]::Create($Path)
  $w = New-Object System.IO.BinaryWriter $fs

  $w.Write([uint16]0)                 # reserved
  $w.Write([uint16]1)                 # type: icon
  $w.Write([uint16]$pngs.Count)

  # Header is 6 bytes, then 16 per directory entry; image data follows all of them.
  $offset = 6 + (16 * $pngs.Count)
  foreach ($p in $pngs) {
    # 256 is stored as 0 — the field is one byte.
    $dim = if ($p.Size -ge 256) { 0 } else { $p.Size }
    $w.Write([byte]$dim); $w.Write([byte]$dim)
    $w.Write([byte]0)                 # palette count
    $w.Write([byte]0)                 # reserved
    $w.Write([uint16]1)               # colour planes
    $w.Write([uint16]32)              # bits per pixel
    $w.Write([uint32]$p.Bytes.Length)
    $w.Write([uint32]$offset)
    $offset += $p.Bytes.Length
  }

  foreach ($p in $pngs) { $w.Write($p.Bytes) }

  $w.Flush(); $w.Close(); $fs.Dispose()
  Write-Host ("{0}  ({1} sizes: {2})" -f (Split-Path $Path -Leaf), $pngs.Count, ($Sizes -join ', '))
}

function Write-Png {
  param($Panel, [int]$Size, [string]$Shape, [string]$Path)
  $bmp = Render -Panel $Panel -Size $Size -Shape $Shape
  $bmp.Save($Path, [System.Drawing.Imaging.ImageFormat]::Png)
  $bmp.Dispose()
  Write-Host ("{0}  ({1}px)" -f (Split-Path $Path -Leaf), $Size)
}

# Panel coordinates, measured off the 1536x1024 sheet.
$app  = Get-Panel -Sheet $Sheet -X 1077 -Y 82  -W 315 -H 315
$tray = Get-Panel -Sheet $Sheet -X 1152 -Y 502 -W 166 -H 166

# The exe and window icon. 256 for Explorer, 32/24 for the taskbar and title bar, 16 for
# the small-icon views that still exist all over the shell.
Write-Ico -Panel $app -Sizes @(16, 24, 32, 48, 64, 128, 256) -Shape 'rounded' -Path (Join-Path $OutDir 'octavia.ico')

# The tray. Windows draws these at 16 and, on a high-DPI display, 20 or 24.
Write-Ico -Panel $tray -Sizes @(16, 20, 24, 32, 48) -Shape 'circle' -Path (Join-Path $OutDir 'octavia-tray.ico')

# For the page the socket serves: a browser is a face since v0.20.0, so a phone that
# shortcuts to her deserves a real icon rather than a globe.
Write-Png -Panel $app -Size 32  -Shape 'rounded' -Path (Join-Path $WebDir 'favicon-32.png')
Write-Png -Panel $app -Size 180 -Shape 'rounded' -Path (Join-Path $WebDir 'apple-touch-icon.png')
Write-Png -Panel $app -Size 192 -Shape 'rounded' -Path (Join-Path $WebDir 'icon-192.png')

$app.Dispose(); $tray.Dispose()
```
