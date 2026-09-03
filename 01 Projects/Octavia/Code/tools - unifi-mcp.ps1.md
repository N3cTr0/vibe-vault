---
project: Octavia
tags: [octavia, code]
source-path: tools\unifi-mcp.ps1
---

# tools\unifi-mcp.ps1

```powershell
<#
.SYNOPSIS
  An MCP server for the UniFi network, over the official local Integration API.

.DESCRIPTION
  The first real tool server in the project, and deliberately the first: it was entirely
  read-only to begin with, so the brain-side tool loop could be written and watched against
  something that had nothing in the house to break.

  It speaks JSON-RPC 2.0 over stdio exactly as `mock-mcp.ps1` does, because that one was
  written to be the shape a real server would take. Eight tools - seven reads and, since
  v0.41.0, **one that changes something**:

    list_devices      the network hardware, with state and firmware
    list_clients      everything connected, which is also how presence is answered
    get_status        gateway load, memory, uptime and current throughput
    find_client       search the connected list by name, address or MAC
    list_ports        switch ports: link, speed, and whether PoE is supplying power
    power_cycle_port  **writes.** Restarts PoE power on one port - off, then on again
    list_cameras      UniFi Protect's cameras and whether they are reachable

  **No Home Assistant required.** The UDM answers this API itself, locally, with a key made
  in its own UI - so network sensing is available a long time before the house is.

  **Network and Protect are one server here, not two**, which is a departure from the
  roadmap's "each integration independently broken-able" and is worth the sentence. They are
  two applications on one appliance, reached at one address with one key: when the UDM is
  unreachable both are, so there is no independence to preserve and a second process would
  buy nothing. Should Protect ever move to its own NVR, splitting this is a copy and a
  config entry.

  Not built: a camera snapshot. Protect will return a JPEG and MCP can carry an image, but
  the brain-side tool loop does not exist yet and neither camera is currently online - two
  blockers, and writing it against neither would be guessing. It is the obvious next tool.

  The output is written for a language model rather than for a parser: short lines of prose
  instead of the raw JSON, because the raw form spends most of its tokens on identifiers
  nothing downstream ever uses.

.NOTES
  Configuration comes from the environment, which is where `McpServer.Env` puts it and why
  that field exists: an argument is visible in the process list to every account on the
  machine.

    UNIFI_API_KEY   required. Settings -> Control Plane -> Integrations, in the UniFi UI.
    UNIFI_HOST      optional, defaults to 10.1.1.1.

  The certificate is the UDM's own self-signed one, so verification is skipped. That is
  safe here and only here: the address is a local gateway on the same wire, reached by IP.
  If this ever points at something across a network, this is the line to revisit.

.EXAMPLE
  $env:UNIFI_API_KEY = '...'; pwsh -NoProfile -File tools\unifi-mcp.ps1
#>
$ErrorActionPreference = 'Stop'

$apiKey = $env:UNIFI_API_KEY
$unifiHost = if ($env:UNIFI_HOST) { $env:UNIFI_HOST } else { '10.1.1.1' }
$base = "https://$unifiHost/proxy/network/integration/v1"
$protect = "https://$unifiHost/proxy/protect/integration/v1"

# Resolved on first use rather than at startup. A server that cannot reach the gateway
# should still start, list its tools, and say what is wrong when one is called - the
# registry logs a failed start as a missing integration, which is a worse diagnosis.
$script:siteId = $null

function Invoke-Unifi([string]$path, [string]$root = $base) {
  if (-not $apiKey) { throw 'UNIFI_API_KEY is not set' }

  Invoke-RestMethod -Uri "$root/$path" -Method Get -SkipCertificateCheck -TimeoutSec 20 `
    -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' }
}

function Get-SiteId {
  if ($script:siteId) { return $script:siteId }
  $sites = Invoke-Unifi 'sites?limit=10'
  if (-not $sites.data -or $sites.data.Count -eq 0) { throw 'the gateway reports no sites' }
  $script:siteId = $sites.data[0].id
  $script:siteId
}

<#
  How long something has been connected, in the units a person would say it in. "Since
  2026-07-28T18:41:59Z" is not an answer to "how long has that been on the network".
#>
function Format-Since([string]$timestamp) {
  if (-not $timestamp) { return 'unknown' }
  try { $span = (Get-Date).ToUniversalTime() - ([datetime]$timestamp).ToUniversalTime() }
  catch { return 'unknown' }

  if ($span.TotalMinutes -lt 60) { return "$([int]$span.TotalMinutes)m" }
  if ($span.TotalHours -lt 48) { return "$([int]$span.TotalHours)h" }
  "$([int]$span.TotalDays)d"
}

function Format-Rate([double]$bitsPerSecond) {
  if ($bitsPerSecond -ge 1e6) { return "{0:N1} Mbps" -f ($bitsPerSecond / 1e6) }
  "{0:N0} kbps" -f ($bitsPerSecond / 1e3)
}

function Get-Devices {
  $site = Get-SiteId
  $devices = (Invoke-Unifi "sites/$site/devices?limit=200").data
  if (-not $devices) { return 'No network hardware is reporting.' }

  $lines = $devices | ForEach-Object {
    "{0} ({1}) - {2}, {3}, firmware {4}" -f `
      $_.name, $_.model, $_.state.ToLower(), $_.ipAddress, $_.firmwareVersion
  }
  ($lines -join "`n")
}

function Get-Clients {
  $site = Get-SiteId
  $clients = (Invoke-Unifi "sites/$site/clients?limit=200").data
  if (-not $clients) { return 'Nothing is connected.' }

  $lines = $clients | Sort-Object name | ForEach-Object {
    $kind = if ($_.type -eq 'WIRELESS') { 'wifi' } else { 'wired' }
    "{0} - {1}, {2}, connected {3}" -f `
      $_.name, $kind, $_.ipAddress, (Format-Since $_.connectedAt)
  }
  "$($clients.Count) connected:`n" + ($lines -join "`n")
}

function Get-Status {
  $site = Get-SiteId
  $devices = (Invoke-Unifi "sites/$site/devices?limit=200").data
  $clients = (Invoke-Unifi "sites/$site/clients?limit=200").data

  $gateway = $devices | Where-Object { $_.features -contains 'switching' } | Select-Object -First 1
  if (-not $gateway) { $gateway = $devices | Select-Object -First 1 }

  $stats = Invoke-Unifi "sites/$site/devices/$($gateway.id)/statistics/latest"
  $up = [timespan]::FromSeconds($stats.uptimeSec)
  $offline = @($devices | Where-Object { $_.state -ne 'ONLINE' })

  $report = @(
    "$($gateway.name): $($gateway.state.ToLower()), up $([int]$up.TotalDays)d $($up.Hours)h"
    "CPU {0:N0}%, memory {1:N0}%, load {2}" -f `
      $stats.cpuUtilizationPct, $stats.memoryUtilizationPct, $stats.loadAverage1Min
    "Throughput: down $(Format-Rate $stats.uplink.rxRateBps), up $(Format-Rate $stats.uplink.txRateBps)"
    "$($devices.Count) network device(s), $($clients.Count) client(s) connected"
  )

  if ($offline.Count -gt 0) {
    $report += "Not online: " + (($offline | ForEach-Object { $_.name }) -join ', ')
  }

  ($report -join "`n")
}

function Find-Client([string]$query) {
  if (-not $query) { return 'Say what to look for.' }

  $site = Get-SiteId
  $clients = (Invoke-Unifi "sites/$site/clients?limit=200").data

  $hits = @($clients | Where-Object {
      "$($_.name) $($_.ipAddress) $($_.macAddress)" -like "*$query*"
    })

  if ($hits.Count -eq 0) { return "Nothing connected matches '$query'." }

  $lines = $hits | ForEach-Object {
    $kind = if ($_.type -eq 'WIRELESS') { 'wifi' } else { 'wired' }
    "{0} - {1}, {2}, {3}, connected {4}" -f `
      $_.name, $kind, $_.ipAddress, $_.macAddress, (Format-Since $_.connectedAt)
  }
  ($lines -join "`n")
}

<#
  The switch ports, and the one thing this server can change.

  **The API does not say what is plugged into a port.** A wired client carries an
  `uplinkDeviceId` - which appliance it hangs off - and no port index, so "port 4 is the
  front door camera" is not a fact available here. That is said out loud in the listing and
  again in the tool description, because the alternative is a model inferring the mapping
  from names and being confidently wrong about which camera it is about to power off.

  So the listing reports what the gateway actually knows: link state, negotiated speed, and
  whether the port supplies PoE and is currently doing so.
#>
function Get-Ports([string]$query) {
  $site = Get-SiteId
  $devices = (Invoke-Unifi "sites/$site/devices?limit=50").data

  $wanted = if ($query) {
    $devices | Where-Object { $_.name -like "*$query*" -or $_.model -like "*$query*" }
  } else {
    $devices
  }

  if (-not $wanted) { return "There is no network device matching '$query'." }

  $lines = @()

  foreach ($device in $wanted) {
    $detail = Invoke-Unifi "sites/$site/devices/$($device.id)"
    $ports = $detail.interfaces.ports | Sort-Object idx
    if (-not $ports) { $lines += "$($device.name): no ports reported."; continue }

    $powered = @($ports | Where-Object { $_.poe }).Count
    $lines += "$($device.name) ($($device.model)): $($ports.Count) ports, $powered of them PoE."

    foreach ($p in $ports) {
      $link = if ($p.state -eq 'UP') { "up at $($p.speedMbps) Mbps" } else { 'nothing linked' }

      $poe = if (-not $p.poe) { 'no PoE' }
             elseif (-not $p.poe.enabled) { "PoE $($p.poe.standard), switched off" }
             elseif ($p.poe.state -eq 'UP') { "PoE $($p.poe.standard), powering something" }
             else { "PoE $($p.poe.standard), on but nothing drawing" }

      $lines += "  port $($p.idx) ($($p.connector)): $link; $poe"
    }
  }

  $lines += ''
  $lines += 'The gateway does not report which client is on which port, so it cannot say what any port feeds.'
  $lines -join "`n"
}

<#
  Power-cycling a PoE port: off, then on again, as one action.

  **POWER_CYCLE is the only thing the API will do to a port**, and that is worth knowing
  rather than working around - it was established by sending a deliberately invalid action
  and reading the refusal, which named the valid set as exactly `POWER_CYCLE`. There is no
  way through this API to switch a port off and leave it off. For a request to kill power to
  something indefinitely, this is not the tool and there isn't one.

  It is a `Confirm` on the brain side: whatever is on the far end of that cable - a camera, an
  access point, a door reader - loses power and reboots, and the gateway will not say what
  that is. `UnifiChecks` pins the classification so that rewording this description cannot
  quietly downgrade it.
#>
function Restart-PortPower([string]$query, $port) {
  if ($null -eq $port) { return 'Which port? The port number is required.' }

  $index = 0
  if (-not [int]::TryParse("$port", [ref]$index)) { return "'$port' is not a port number." }

  $site = Get-SiteId
  $devices = (Invoke-Unifi "sites/$site/devices?limit=50").data

  # No device named: the one that actually has PoE ports. With a single such appliance that
  # is unambiguous; with two it asks rather than guessing, because guessing wrong here cuts
  # power to something in another room.
  $candidates = @()
  foreach ($device in $devices) {
    if ($query -and $device.name -notlike "*$query*" -and $device.model -notlike "*$query*") { continue }
    $detail = Invoke-Unifi "sites/$site/devices/$($device.id)"
    if ($detail.interfaces.ports | Where-Object { $_.poe }) {
      $candidates += [pscustomobject]@{ Device = $device; Detail = $detail }
    }
  }

  # Written as an assignment rather than `return if (...)`, which *parses* - PowerShell reads
  # `if` there as a command name - and then throws at runtime. See Lessons Learned.
  if ($candidates.Count -eq 0) {
    $answer = if ($query) { "No PoE device matches '$query'." } else { 'No device here supplies PoE.' }
    return $answer
  }
  if ($candidates.Count -gt 1) {
    return "More than one device supplies PoE ($(($candidates.Device.name) -join ', ')). Say which one."
  }

  $chosen = $candidates[0]
  $target = $chosen.Detail.interfaces.ports | Where-Object { $_.idx -eq $index }

  if (-not $target) { return "$($chosen.Device.name) has no port $index." }
  if (-not $target.poe) { return "Port $index on $($chosen.Device.name) does not supply PoE, so there is no power to cycle." }

  # What it looked like before, so the answer can be compared against something rather than
  # just asserting success.
  $before = if ($target.state -eq 'UP') { "was linked at $($target.speedMbps) Mbps" } else { 'had nothing linked' }

  Invoke-RestMethod -Uri "$base/sites/$site/devices/$($chosen.Device.id)/interfaces/ports/$index/actions" `
    -Method Post -SkipCertificateCheck -TimeoutSec 30 `
    -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
    -ContentType 'application/json' -Body '{"action":"POWER_CYCLE"}' | Out-Null

  "Power-cycled port $index on $($chosen.Device.name). It $before. Whatever is on it has lost " +
  "power and is starting again - give it a minute before asking whether it is back."
}

<#
  Protect answers on the same appliance, to the same key, with a flat array rather than the
  Network API's paged envelope - so this one reads `$cameras` directly and not `.data`.

  A camera that is present but unreachable is the case worth naming out loud. Both of these
  read DISCONNECTED and a snapshot returns 503, so the state is real rather than stale, and
  "she has cameras" and "she can see anything" are separate claims.
#>
function Get-Cameras {
  $cameras = Invoke-Unifi 'cameras' $protect
  if (-not $cameras -or $cameras.Count -eq 0) { return 'There are no cameras.' }

  $lines = $cameras | Sort-Object name | ForEach-Object {
    $reachable = if ($_.state -eq 'CONNECTED') { 'online' } else { "$($_.state.ToLower()) - not reachable" }
    "{0} ({1}) - {2}" -f $_.name, $_.type, $reachable
  }

  $up = @($cameras | Where-Object { $_.state -eq 'CONNECTED' }).Count
  "$($cameras.Count) camera(s), $up online:`n" + ($lines -join "`n")
}

<#
  A snapshot, as an MCP image block.

  Two refusals rather than one, because they are different problems with different answers:
  a camera nobody has heard of is a typo, and a camera that is present but unreachable is a
  fact about the house. Saying "not reachable" for both would send her looking for a name
  that was never wrong.

  `highQuality` is asked for and *not* required. This G5 Bullet answers
  "Camera does not support full HD snapshot" with a 400, so the standard frame is fetched
  instead - written this way because the next camera may well support it and nobody should
  have to come back here.
#>
function Get-CameraView([string]$query) {
  if (-not $query) { return @{ text = 'Say which camera to look through.' } }

  $cameras = Invoke-Unifi 'cameras' $protect
  $hit = @($cameras | Where-Object { $_.name -like "*$query*" }) | Select-Object -First 1

  if (-not $hit) {
    $names = ($cameras | ForEach-Object { $_.name }) -join ', '
    return @{ text = "There is no camera called '$query'. There is: $names." }
  }

  if ($hit.state -ne 'CONNECTED') {
    return @{ text = "The $($hit.name) camera is $($hit.state.ToLower()) and cannot be reached, so there is nothing to see through it." }
  }

  $url = "$protect/cameras/$($hit.id)/snapshot"
  $bytes = $null

  foreach ($attempt in @("$($url)?highQuality=true", $url)) {
    try {
      $bytes = Invoke-WebRequest -Uri $attempt -Method Get -SkipCertificateCheck -TimeoutSec 30 `
        -Headers @{ 'X-API-KEY' = $apiKey } | Select-Object -ExpandProperty Content
      break
    }
    catch { continue }
  }

  if (-not $bytes) { return @{ text = "The $($hit.name) camera did not return a picture." } }

  @{
    text  = "Looking through the $($hit.name) camera, just now."
    image = [Convert]::ToBase64String($bytes)
  }
}

<#
  Descriptions are read by the risk heuristic in `McpClient.RiskOf` as well as by the
  model, and it checks its dangerous words first. Every one of these is a read and must
  classify as one, so the wording deliberately avoids the vocabulary that would make it
  ask permission - there is nothing here worth asking about.
#>
$tools = @(
  @{
    name        = 'list_devices'
    description = 'List the UniFi network hardware - gateway and access points - with model, state, address and firmware version.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'list_clients'
    description = 'List everything currently connected to the network, wired and wireless, with name, address and how long it has been connected. This is how to answer who or what is at home.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'get_status'
    description = 'Read overall network health: gateway load, memory, uptime, current throughput, and how many devices and clients are present.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'find_client'
    description = 'Search the connected clients by name, IP address or MAC and report what matches.'
    inputSchema = @{
      type       = 'object'
      properties = @{ query = @{ type = 'string'; description = 'Name, address or MAC, or part of one' } }
      required   = @('query')
    }
  },
  @{
    name        = 'list_ports'
    description = 'Read the switch ports on a network device: whether each one has a link, how fast, and whether it supplies PoE power and is currently doing so. The gateway does not report which client is on which port, so this cannot say what a port feeds.'
    inputSchema = @{
      type       = 'object'
      properties = @{ device = @{ type = 'string'; description = 'Device name or part of one. Omitted means every device.' } }
    }
  },
  @{
    name        = 'power_cycle_port'
    description = 'Restart the PoE power on one switch port - it goes off and comes back on again. Whatever is plugged into that port loses power and reboots, and the gateway cannot say what that is. The port cannot be left switched off; off-and-on is the only thing the hardware will do.'
    inputSchema = @{
      type       = 'object'
      properties = @{
        port   = @{ type = 'integer'; description = 'Port number, as shown by list_ports' }
        device = @{ type = 'string'; description = 'Device name or part of one. Omitted picks the only device that supplies PoE.' }
      }
      required   = @('port')
    }
  },
  @{
    name        = 'list_cameras'
    description = 'List the UniFi Protect cameras and whether each one is currently reachable. Use this to say what she can and cannot see.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'look_at_camera'
    description = 'Look through one UniFi Protect camera and see what is there now. Give the camera name, or part of it. Returns a picture.'
    inputSchema = @{
      type       = 'object'
      properties = @{ camera = @{ type = 'string'; description = 'Camera name, or part of one, such as Front Door' } }
      required   = @('camera')
    }
  }
)

function Send-Message($object) {
  # -Compress keeps it to one line, which the stdio framing requires.
  [Console]::Out.WriteLine(($object | ConvertTo-Json -Depth 12 -Compress))
  [Console]::Out.Flush()
}

while ($true) {
  $line = [Console]::In.ReadLine()
  if ($null -eq $line) { break }
  if ($line.Trim().Length -eq 0) { continue }

  try { $message = $line | ConvertFrom-Json } catch { continue }

  # A notification has no id and wants no answer.
  if ($null -eq $message.id) { continue }

  switch ($message.method) {
    'initialize' {
      Send-Message @{
        jsonrpc = '2.0'; id = $message.id
        result  = @{
          protocolVersion = '2025-06-18'
          capabilities    = @{ tools = @{} }
          serverInfo      = @{ name = 'octavia-unifi'; version = '1.0.0' }
        }
      }
    }

    'tools/list' {
      Send-Message @{ jsonrpc = '2.0'; id = $message.id; result = @{ tools = $tools } }
    }

    'tools/call' {
      $name = $message.params.name
      $callArgs = $message.params.arguments

      # Failures come back as text rather than as a JSON-RPC error, because the seam says
      # so and the reason is good: a model told "the gateway did not answer" can say that
      # out loud, where an error only ends the turn with nothing to relay.
      $picture = $null

      try {
        $text = switch ($name) {
          'list_devices' { Get-Devices }
          'list_clients' { Get-Clients }
          'get_status' { Get-Status }
          'find_client' { Find-Client $callArgs.query }
          'list_ports' { Get-Ports $callArgs.device }
          'power_cycle_port' { Restart-PortPower $callArgs.device $callArgs.port }
          'list_cameras' { Get-Cameras }
          'look_at_camera' {
            $seen = Get-CameraView $callArgs.camera
            $picture = $seen.image
            $seen.text
          }
          default { "No such tool: $name" }
        }
      }
      catch {
        $text = "The UniFi gateway could not be reached: $($_.Exception.Message)"
      }

      # The words always, the picture only when there is one - and the text first, so a
      # reader of the raw protocol sees what happened before several hundred KB of base64.
      $content = @(@{ type = 'text'; text = $text })
      if ($picture) {
        $content += @{ type = 'image'; data = $picture; mimeType = 'image/jpeg' }
      }

      Send-Message @{
        jsonrpc = '2.0'; id = $message.id
        result  = @{ content = $content }
      }
    }

    default {
      Send-Message @{
        jsonrpc = '2.0'; id = $message.id
        error   = @{ code = -32601; message = "Method not found: $($message.method)" }
      }
    }
  }
}
```
