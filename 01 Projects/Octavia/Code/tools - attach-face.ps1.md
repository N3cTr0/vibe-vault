---
project: Octavia
tags: [octavia, code]
source-path: tools\attach-face.ps1
---

# tools\attach-face.ps1

```powershell
<#
.SYNOPSIS
  Attach to a running Octavia as an external face.

.DESCRIPTION
  Proof that the protocol, not the WebView2 page, is the interface: this script is a
  face. It connects over the same socket the built-in page uses, prints every message
  she sends, and can speak to her.

  Reads the port and token from octavia.log, which is where the host writes them at
  startup. She must be running.

.PARAMETER Say
  Send this text as if typed into her console, then print what comes back.

.PARAMETER Send
  Raw protocol messages to send, as JSON. Anything in PROTOCOL.md's face-to-host table
  works; this is how a message with no dedicated switch gets exercised.

.PARAMETER Conformance
  Drive her through a scripted turn and report which host-to-face messages arrived and
  whether each carried the fields PROTOCOL.md promises.

  This is the question Stage 8 rests on: a photoreal renderer is only a *swap* if the
  host already says everything such a renderer needs to hear. Run it before building a
  face, and against the host after changing one. Exit code is the number of failures.

.PARAMETER Seconds
  How long to listen before disconnecting.

.EXAMPLE
  pwsh -File tools\attach-face.ps1
.EXAMPLE
  pwsh -File tools\attach-face.ps1 -Say "what are you?"
.EXAMPLE
  pwsh -File tools\attach-face.ps1 -Conformance
#>
[CmdletBinding()]
param(
  [string]$Say,
  [string[]]$Send,
  [switch]$Conformance,
  [int]$Seconds = 20
)

$ErrorActionPreference = 'Stop'

# Her data folder moved into the repo in v0.11.0, so the log is looked for the same way
# Core\Paths.cs resolves it: OCTAVIA_DATA, then <repo>\data, then %APPDATA%. An older
# install still keeps it in the last of those, which is why all three are tried.
$candidates = @(
  $(if ($env:OCTAVIA_DATA) { Join-Path $env:OCTAVIA_DATA 'octavia.log' })
  (Join-Path (Split-Path $PSScriptRoot -Parent) 'data\octavia.log')
  (Join-Path $env:APPDATA 'Octavia\octavia.log')
) | Where-Object { $_ }

$log = $candidates | Where-Object { Test-Path -LiteralPath $_ } | Select-Object -First 1
if (-not $log) { throw "No log found in any of:`n  $($candidates -join "`n  ")`nIs she running?" }

$line = Select-String -Path $log -Pattern 'face socket listening on (ws://\S+)' |
  Select-Object -Last 1
if (-not $line) { throw 'No face socket in the log. Is she running, and did the port bind?' }

$url = $line.Matches[0].Groups[1].Value
Write-Host "attaching to $url"

$socket = [System.Net.WebSockets.ClientWebSocket]::new()
$socket.ConnectAsync([Uri]$url, [Threading.CancellationToken]::None).GetAwaiter().GetResult() | Out-Null
Write-Host "attached as an external face`n"

function Send-Face([string]$json) {
  $bytes = [Text.Encoding]::UTF8.GetBytes($json)
  $segment = [ArraySegment[byte]]::new($bytes)
  $socket.SendAsync($segment, 'Text', $true, [Threading.CancellationToken]::None).GetAwaiter().GetResult() | Out-Null
}

<#
  What a renderer has to be told, and what each message must carry to be usable.

  `Required` marks the messages a face cannot perform without, and which an ordinary
  turn must therefore produce. The rest depend on conditions this script cannot create
  on demand — there may be no music playing, and she may already have a key — so they
  are reported as *not observed* rather than as failures. Saying "the host did not send
  this" when the truth is "the situation did not arise" would be a worse report than none.
#>
$contract = [ordered]@{
  hello       = @{ Required = $true;  Fields = @('protocol', 'model', 'profile', 'ears', 'voice', 'voiceEngine', 'listening', 'roomHour', 'state', 'emotion', 'emotionWeight') }
  state       = @{ Required = $true;  Fields = @('value'); OneOf = @{ value = @('idle', 'listening', 'thinking', 'speaking') } }
  caption     = @{ Required = $true;  Fields = @('who', 'text') }
  turn        = @{ Required = $true;  Fields = @('who', 'text'); OneOf = @{ who = @('you', 'octavia') } }
  viseme      = @{ Required = $true;  Fields = @('value') }
  # Conditional, not required: it is sent only when her mood *changes*, and a reply can
  # perfectly well be neutral throughout. What a renderer actually needs is the current
  # expression at attach time, which `hello` carries.
  emotion     = @{ Required = $false; Fields = @('value', 'weight'); OneOf = @{ value = @('neutral', 'happy', 'angry', 'sad', 'relaxed', 'surprised') } }
  diagnostics = @{ Required = $true;  Fields = @() }
  cleared     = @{ Required = $true;  Fields = @() }
  level       = @{ Required = $false; Fields = @('value') }
  music       = @{ Required = $false; Fields = @() }
  notice      = @{ Required = $false; Fields = @('text') }
  needKey     = @{ Required = $false; Fields = @() }
}

$seen = @{}
$problems = @()

function Record($msg) {
  $type = $msg.type
  if (-not $type) { return }

  $seen[$type] = ($seen[$type] ?? 0) + 1
  $rule = $contract[$type]
  if (-not $rule) { return }

  # Only the first of a repeating message is inspected; a hundred identical complaints
  # about `viseme` would bury the one that mattered.
  if ($seen[$type] -gt 1) { return }

  foreach ($field in $rule.Fields) {
    if ($null -eq $msg.PSObject.Properties[$field]) {
      $script:problems += "$type is missing '$field'"
    }
  }

  if ($rule.OneOf) {
    foreach ($field in $rule.OneOf.Keys) {
      $value = $msg.$field
      if ($null -ne $value -and $value -notin $rule.OneOf[$field]) {
        $script:problems += "$type.$field was '$value', outside the documented vocabulary"
      }
    }
  }
}

Send-Face '{"type":"ready","faceBuilt":true}'

if ($Conformance) {
  # A scripted turn, then the two messages a turn does not produce. `saveDiagnostics` is
  # deliberately never sent: it raises a file dialog on the host, and a check that needs
  # somebody to click Cancel is not a check.
  Write-Host "driving a turn, a self-test and a forget...`n"
  Send-Face '{"type":"selfTest"}'
  Send-Face (@{ type = 'say'; text = 'In one short sentence, say hello.' } | ConvertTo-Json -Compress)
} else {
  if ($Say) { Send-Face (@{ type = 'say'; text = $Say } | ConvertTo-Json -Compress) }
  foreach ($message in $Send) { Send-Face $message }
}

$buffer = [ArraySegment[byte]]::new([byte[]]::new(16384))

# One token for the whole session. Cancelling an individual ReceiveAsync ABORTS the
# socket rather than timing out, so a per-read timeout would kill the connection on
# the first quiet moment.
$window = if ($Conformance) { [Math]::Max($Seconds, 45) } else { $Seconds }
$session = [Threading.CancellationTokenSource]::new([TimeSpan]::FromSeconds($window))
$timedOut = $false
$forgotten = $false

while ($socket.State -eq 'Open' -and -not $session.IsCancellationRequested) {
  try {
    $result = $socket.ReceiveAsync($buffer, $session.Token).GetAwaiter().GetResult()
  } catch {
    $timedOut = $true
    break
  }

  if ($result.Count -eq 0) { continue }
  $text = [Text.Encoding]::UTF8.GetString($buffer.Array, 0, $result.Count)
  $msg  = $text | ConvertFrom-Json

  Record $msg

  if ($Conformance) {
    # `cleared` only comes back for a forget, and forgetting mid-reply would cut the
    # turn short — so it waits until she has finished saying something.
    if (-not $forgotten -and $msg.type -eq 'turn' -and $msg.who -eq 'octavia') {
      $forgotten = $true
      Send-Face '{"type":"forget"}'
    }
    if ($msg.type -notin @('level', 'viseme')) { Write-Host -NoNewline "$($msg.type) " }
    continue
  }

  # level and viseme arrive many times a second; summarise rather than flood.
  switch ($msg.type) {
    'level'   { Write-Host -NoNewline '.' }
    'viseme'  { Write-Host -NoNewline '~' }
    'hello'   {
      Write-Host "hello    protocol $($msg.protocol), brain $($msg.model), profile $($msg.profile), ears $($msg.ears)"
      # The devices she is actually using, which is the question whenever she cannot
      # hear you or cannot hear the music. An empty value means the Windows default.
      $mic = if ($msg.microphone) { $msg.microphone } else { 'Windows default' }
      $out = if ($msg.output) { $msg.output } else { 'Windows default' }
      Write-Host "         mic: $mic  |  output: $out  |  whisper on: $($msg.whisperCompute)"
      Write-Host "         mics available: $(($msg.microphones | ForEach-Object { $_.label }) -join '; ')"
      Write-Host "         outputs available: $(($msg.outputs | ForEach-Object { $_.label }) -join '; ')"
    }
    'state'   { Write-Host "state    $($msg.value)" }
    'caption' { Write-Host "caption  [$($msg.who)] $($msg.text)" }
    'turn'    { Write-Host "turn     [$($msg.who)] $($msg.text)" }
    'notice'  { Write-Host "notice   $($msg.text)" }
    'diagnostics' {
      if ($msg.running) { Write-Host 'selftest running...'; break }
      foreach ($check in $msg.checks) {
        Write-Host ("  [{0}] {1}: {2}" -f $(if ($check.ok) { ' ok ' } else { 'FAIL' }), $check.name, $check.detail)
        if (-not $check.ok -and $check.fix) { Write-Host "         -> $($check.fix)" }
      }
      foreach ($fact in $msg.facts) { Write-Host ("  {0,-18} {1}" -f $fact.name, $fact.value) }
    }
    default   { Write-Host "$($msg.type)" }
  }

  # Nothing more is coming after the transcript line for her reply.
  if (-not $Conformance -and $msg.type -eq 'turn' -and $msg.who -eq 'octavia' -and -not $Send) { break }
}

# An aborted socket cannot be closed politely; only shake hands if it is still open.
if (-not $timedOut -and $socket.State -eq 'Open') {
  $socket.CloseAsync('NormalClosure', 'done', [Threading.CancellationToken]::None).GetAwaiter().GetResult() | Out-Null
}
$socket.Dispose()
$session.Dispose()

if (-not $Conformance) {
  Write-Host "`ndetached"
  return
}

Write-Host "`n`nrenderer contract"
Write-Host    "-----------------"

$missing = 0
foreach ($type in $contract.Keys) {
  $count = $seen[$type] ?? 0
  $rule = $contract[$type]

  $verdict = if ($count -gt 0) { 'ok  ' }
             elseif ($rule.Required) { $missing++; 'MISS' }
             else { '  - ' }

  $note = if ($count -gt 0) { "$count received" }
          elseif ($rule.Required) { 'never arrived' }
          else { 'not observed (conditions did not arise)' }

  Write-Host ("  [{0}] {1,-12} {2}" -f $verdict, $type, $note)
}

# Anything she sent that is not written down. Not a failure — the protocol is additive
# by design — but a renderer author would want to know it exists.
$undocumented = $seen.Keys | Where-Object { -not $contract[$_] } | Sort-Object
if ($undocumented) {
  Write-Host "`n  undocumented message types: $($undocumented -join ', ')"
}

if ($problems) {
  Write-Host "`nfield problems"
  Write-Host   "--------------"
  $problems | Sort-Object -Unique | ForEach-Object { Write-Host "  $_" }
}

$failures = $missing + ($problems | Sort-Object -Unique).Count
Write-Host ""
Write-Host $(if ($failures -eq 0) {
  'CONTRACT MET - the host already tells a renderer everything it needs.'
} else {
  "$failures PROBLEM(S) - a renderer could not be written against this host as it stands."
})

exit $failures
```
