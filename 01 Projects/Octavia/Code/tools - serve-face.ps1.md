---
project: Octavia
tags: [octavia, code]
source-path: tools\serve-face.ps1
---

# tools\serve-face.ps1

```powershell
<#
    Serves wwwroot over loopback so the face can be opened in an ordinary browser.

    The face normally lives inside WebView2, where looking at it costs a rebuild, a
    relaunch and a screenshot. Served here it has devtools, live reload by refresh, and
    `window.Face` on the console — so a renderer change can be judged in seconds:

        window.Face.setMusic({ playing: true, bpm: 128, energy: 0.8 })
        setInterval(() => window.Face.setMusic({ beat: true }), 60000 / 128)

    Nothing else works: there is no host, so it reports "No host to connect to" and every
    control is inert. That is the point — this is for the renderer, not the app.

    A raw TcpListener rather than HttpListener, for the same reason the face socket is
    one: HttpListener wants a urlacl reservation, and this should never need elevation.

    Stop it with Ctrl+C. It binds 127.0.0.1 only.
#>
param(
    [int]$Port = 8999,
    [string]$Root = (Join-Path $PSScriptRoot '..\src\Octavia.App\wwwroot')
)

$ErrorActionPreference = 'Stop'
$Root = (Resolve-Path $Root).Path

$types = @{
    '.html' = 'text/html; charset=utf-8'
    '.js'   = 'text/javascript; charset=utf-8'
    '.css'  = 'text/css; charset=utf-8'
    '.json' = 'application/json; charset=utf-8'
    '.vrm'  = 'application/octet-stream'
    '.png'  = 'image/png'
    '.svg'  = 'image/svg+xml'
}

$listener = [Net.Sockets.TcpListener]::new([Net.IPAddress]::Loopback, $Port)
$listener.Start()
Write-Host "serving $Root at http://127.0.0.1:$Port/index.html  (Ctrl+C to stop)"

try {
    while ($true) {
        $client = $listener.AcceptTcpClient()
        try {
            $stream = $client.GetStream()
            $reader = [IO.StreamReader]::new($stream)
            $request = $reader.ReadLine()
            if (-not $request) { continue }

            $path = ($request -split ' ')[1]
            $path = ($path -split '\?')[0]
            if ($path -eq '/') { $path = '/index.html' }

            # Everything is resolved under the root and then checked to still be under
            # it, so a request full of ".." cannot walk out into the repo.
            $file = Join-Path $Root ($path.TrimStart('/') -replace '/', '\')
            $full = [IO.Path]::GetFullPath($file)

            if (-not $full.StartsWith($Root, 'OrdinalIgnoreCase') -or -not (Test-Path $full -PathType Leaf)) {
                $body = [Text.Encoding]::UTF8.GetBytes('not found')
                $head = "HTTP/1.1 404 Not Found`r`nContent-Length: $($body.Length)`r`nConnection: close`r`n`r`n"
            }
            else {
                $body = [IO.File]::ReadAllBytes($full)
                $type = $types[[IO.Path]::GetExtension($full).ToLowerInvariant()]
                if (-not $type) { $type = 'application/octet-stream' }
                $head = "HTTP/1.1 200 OK`r`nContent-Type: $type`r`nContent-Length: $($body.Length)`r`nCache-Control: no-store`r`nConnection: close`r`n`r`n"
            }

            $headBytes = [Text.Encoding]::ASCII.GetBytes($head)
            $stream.Write($headBytes, 0, $headBytes.Length)
            $stream.Write($body, 0, $body.Length)
            $stream.Flush()
        }
        catch {
            Write-Warning $_.Exception.Message
        }
        finally {
            $client.Close()
        }
    }
}
finally {
    $listener.Stop()
    Write-Host 'stopped'
}
```
