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
  written to be the shape a real server would take. Thirteen tools - nine reads and, since
  v0.41.0, **four that change something**:

    list_devices         the network hardware, with state and firmware
    list_clients         everything connected, which is also how presence is answered
    get_status           gateway load, memory, uptime and current throughput
    find_client          search the connected list by name, address or MAC
    list_ports           switch ports: link, speed, and whether PoE is supplying power
    list_firewall_rules  which zones reach which, and anything added by hand
    recent_threats       the security log, grouped by which client set it off
    list_cameras         UniFi Protect's cameras and whether they are reachable
    look_at_camera       a still from one camera, as a picture

    power_cycle_port     **writes.** Restarts PoE power on one port - off, then on again
    set_port_power       **writes.** Switches one port's PoE off, or on, and leaves it there
    restart_device       **writes.** Reboots an access point, or the gateway itself
    set_client_access    **writes.** Lets one client onto the network, or takes it off

  **What the API will do, established by asking it rather than reading about it.** Sending a
  deliberately invalid action makes the gateway name the valid set, which is how all three of
  these were settled and how they stay settled when the firmware moves:

    a port    POWER_CYCLE, and nothing else
    a device  RESTART, and nothing else
    a client  AUTHORIZE_GUEST_ACCESS, UNAUTHORIZE_GUEST_ACCESS

  The firewall is **read and never written**. Reading it is nearly all of the value and none
  of the risk: a wrong sentence about a rule costs a sentence, and a wrong change to one can
  take the house off the internet from another room with no undo she can offer.

  **Two APIs, not one.** Everything reads through the official local Integration API. The one
  thing that API cannot do is leave a port switched off - its only port action is
  `POWER_CYCLE`, confirmed by asking for eight others and reading the refusals - so
  `set_port_power` reaches the older `/proxy/network/api` that the UniFi web UI itself uses.
  Same appliance, same key, older contract; see `Set-PortPower`.

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

# The older API names its site "default" where the integration API uses a UUID. Resolved
# on first write, and only then, because nothing that only reads ever needs it.
$script:legacySite = $null

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
<#
  The security log, which the API key cannot reach at all.

  **Two different APIs on one appliance, and one credential between them.** Everything above
  uses the official Integration API. That API is inventory and control: it has no history of
  any kind, and every event-shaped route on it 404s - established with a nonsense-path
  control, so those are real absences rather than the proxy refusing before it routes.

  The events live in the *legacy* API, the one the UniFi web UI itself calls.

  > **This used to log in with a username and password**, on the belief that the older API
  > was behind a cookie session and the key could not reach it. Half of that was true - it
  > *accepts* a cookie session - and the untested half was wrong: it accepts `X-API-KEY` just
  > as readily, on reads and on writes. So the second credential bought nothing, and what it
  > cost was a real UniFi *account* password sitting in the secret store, which is worth more
  > to an attacker than the key is. Removed in v0.49.0, along with the session, the CSRF
  > token and the 401 retry that existed only to renew them.

  `UNIFI_USERNAME` and `UNIFI_PASSWORD` are no longer read by anything here. The stored
  password can be deleted, and the read-only account with it.
#>
function Invoke-Legacy([string]$path, $body) {
  if (-not $apiKey) { throw 'UNIFI_API_KEY is not set' }

  Invoke-RestMethod -Uri "https://$unifiHost$path" -Method Post -SkipCertificateCheck -TimeoutSec 30 `
    -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
    -ContentType 'application/json' -Body ($body | ConvertTo-Json -Compress)
}

<#
  What the security log holds since a moment, grouped by who set it off.

  **`format`** decides who the answer is for. `words` is prose for her to read out; `counts`
  is one `name<TAB>number` per line, which is what `ThreatRound` parses. Two shapes rather
  than a model being asked to count, and rather than a round being asked to read English.

  The severity is *not* a filter, and that is the finding this tool was built around: every
  one of the 195 security events in the first eight-hour sample read `VERY_HIGH`. A threshold
  on severity would have her talking every hour for ever. What matters is whether the pattern
  changed, which is `Baseline`'s job and not this one's - so this reports, and judges nothing.
#>
function Get-Threats([string]$format, $sinceMs) {
  $since = if ($sinceMs) { [int64]$sinceMs } else { [DateTimeOffset]::UtcNow.AddHours(-1).ToUnixTimeMilliseconds() }

  $rows = @()
  $page = 0

  # Paged until the rows are older than asked for. A busy network can produce hundreds an
  # hour, and the endpoint has no "since" of its own.
  while ($page -lt 10) {
    $answer = Invoke-Legacy '/proxy/network/v2/api/site/default/system-log/all' `
      @{ pageNumber = $page; pageSize = 200 }

    $batch = @($answer.data)
    if ($batch.Count -eq 0) { break }

    $rows += $batch | Where-Object { $_.category -eq 'SECURITY' -and $_.timestamp -ge $since }
    if (($batch | Measure-Object -Property timestamp -Minimum).Minimum -lt $since) { break }

    $page++
  }

  # Who set it off. A named client when the gateway knows one, the bare address when it does
  # not - and never blank, because a blank key would silently merge unrelated sources into
  # one row and hide exactly the new thing this exists to notice.
  $named = $rows | ForEach-Object {
    $who = $_.parameters.SRC_CLIENT.name
    if (-not $who) { $who = $_.parameters.SRC_CLIENT.hostname }
    if (-not $who) { $who = $_.parameters.SRC_IP.id }
    if (-not $who) { $who = 'unattributed' }
    [pscustomobject]@{ Who = $who }
  }

  $groups = $named | Group-Object Who | Sort-Object Count -Descending

  if ($format -eq 'counts') {
    $lines = @("total`t$($rows.Count)")
    foreach ($g in $groups) { $lines += "$($g.Name)`t$($g.Count)" }
    return ($lines -join "`n")
  }

  $when = [DateTimeOffset]::FromUnixTimeMilliseconds($since).LocalDateTime.ToString('HH:mm')

  if ($rows.Count -eq 0) { return "No security events since $when." }

  $lines = @("$($rows.Count) security event(s) since $when, all blocked by Threat Management:")
  foreach ($g in $groups) { $lines += "  $($g.Name): $($g.Count)" }
  $lines += 'The log does not name which rule matched; that is only on the Threat Management screen.'
  $lines -join "`n"
}

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
  Which device supplies PoE, and its port detail - the question `list_ports`,
  `power_cycle_port` and `set_port_power` all had to answer separately.

  No device named: the one that actually has PoE ports. With a single such appliance that
  is unambiguous; with two it asks rather than guessing, because guessing wrong here cuts
  power to something in another room.

  Returns either a `[pscustomobject]` with `Device`/`Detail`, or a string, which the caller
  hands straight back to her. "Answer or explanation" rather than an exception: every other
  failure in this file is text she can read out, and this one is no different.
#>
function Resolve-PoeDevice([string]$query, [string]$site) {
  $devices = (Invoke-Unifi "sites/$site/devices?limit=50").data

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

  $candidates[0]
}

<#
  How one PoE port reads right now.

  `poe.enabled` is the setting and `poe.state` is the draw, and they are not the same
  question: a port switched on with nothing plugged in reads enabled with state DOWN. The
  draw also lags the setting by a few seconds, so anything verifying a *switch* watches
  `enabled` and anything verifying a *cycle* watches `state`.
#>
function Read-Port([string]$site, [string]$deviceId, [int]$index) {
  $detail = Invoke-Unifi "sites/$site/devices/$deviceId"
  $detail.interfaces.ports | Where-Object { $_.idx -eq $index }
}

<#
  The same device, as the older API sees it - which is where `port_overrides` lives.

  The two APIs do not share identifiers: the integration API's device id is a UUID and the
  older one's `_id` is a Mongo id, and nothing maps between them but the MAC. So this
  matches on that, and refuses rather than guesses when it cannot.
#>
function Get-LegacyDevice($device) {
  if (-not $script:legacySite) {
    $sites = Invoke-RestMethod -Uri "https://$unifiHost/proxy/network/api/self/sites" `
      -SkipCertificateCheck -TimeoutSec 20 `
      -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' }
    if (-not $sites.data) { return 'The gateway did not list any sites on its own API.' }
    $script:legacySite = $sites.data[0].name
  }

  $all = Invoke-RestMethod -Uri "https://$unifiHost/proxy/network/api/s/$($script:legacySite)/stat/device" `
    -SkipCertificateCheck -TimeoutSec 30 `
    -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' }

  $mac = "$($device.macAddress)".Replace('-', ':').ToLowerInvariant()
  $match = $all.data | Where-Object { "$($_.mac)".ToLowerInvariant() -eq $mac }

  if (-not $match) {
    return "The gateway's own API does not list $($device.name), so its ports cannot be changed from here."
  }

  [pscustomobject]@{ Site = $script:legacySite; Device = $match }
}

<#
  Power-cycling a PoE port: off, then on again, as one action.

  **POWER_CYCLE is the only thing the *integration* API will do to a port**, established by
  sending a deliberately invalid action and reading the refusal, which named the valid set as
  exactly `POWER_CYCLE`. Re-confirmed in v0.49.0 against seven more guesses; the answer has
  not moved. Leaving a port off is `set_port_power`, which reaches another API to do it.

  It is a `Confirm` on the brain side: whatever is on the far end of that cable - a camera, an
  access point, a door reader - loses power and reboots, and the gateway will not say what
  that is. `UnifiChecks` pins the classification so that rewording this description cannot
  quietly downgrade it.

  **The answer is read back rather than asserted.** Until v0.49.0 this posted the action,
  piped the response to `Out-Null` and reported success - so a refusal that did not throw,
  and every failure short of an exception, read as "done". The port is watched afterwards and
  the reply says what was actually seen.
#>
function Restart-PortPower([string]$query, $port) {
  if ($null -eq $port) { return 'Which port? The port number is required.' }

  $index = 0
  if (-not [int]::TryParse("$port", [ref]$index)) { return "'$port' is not a port number." }

  $site = Get-SiteId
  $chosen = Resolve-PoeDevice $query $site
  if ($chosen -is [string]) { return $chosen }

  $target = $chosen.Detail.interfaces.ports | Where-Object { $_.idx -eq $index }

  if (-not $target) { return "$($chosen.Device.name) has no port $index." }
  if (-not $target.poe) {
    return "Port $index on $($chosen.Device.name) does not supply PoE, so there is no power to cycle."
  }
  if (-not $target.poe.enabled) {
    return "Port $index on $($chosen.Device.name) has its PoE switched off, so there is no power to cycle. " +
           'Switch it back on first.'
  }

  <# **Enabled is not the same as supplying**, and the gateway is stricter than it reads.
     A port that is switched on but has nothing drawing from it - empty socket, or a device
     still coming up after being switched back on - answers a POWER_CYCLE with a bare 422.
     Asking first turns that into a sentence worth hearing. #>
  if ($target.poe.state -ne 'UP') {
    return "Port $index on $($chosen.Device.name) is switched on but nothing is drawing power from it, " +
           'so there is nothing to restart.'
  }

  # What it looked like before, so the answer can be compared against something rather than
  # just asserting success.
  $before = if ($target.state -eq 'UP') { "was linked at $($target.speedMbps) Mbps" } else { 'had nothing linked' }

  <# `Invoke-WebRequest` throws on any 4xx or 5xx, so a status check after the call is dead
     code - which is exactly what the first draft of this contained. The refusal has to be
     caught to be read, and `-SkipHttpErrorCheck` would hand back the body without the
     exception at the cost of checking every status by hand. #>
  try {
    Invoke-WebRequest -Uri "$base/sites/$site/devices/$($chosen.Device.id)/interfaces/ports/$index/actions" `
      -Method Post -SkipCertificateCheck -TimeoutSec 30 `
      -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
      -ContentType 'application/json' -Body '{"action":"POWER_CYCLE"}' | Out-Null
  }
  catch {
    $code = $_.Exception.Response.StatusCode.value__
    return "The gateway refused to cycle port $index on $($chosen.Device.name) (HTTP $code). " +
           'Nothing on it was restarted.'
  }

  <# The draw usually drops inside two seconds, and watching for it is the difference between
     "the request was accepted" and "the power actually went off".

     **Twelve attempts rather than six**, because a gateway asked to cycle the same port
     several times in a few minutes takes noticeably longer to act on it, and a window that
     is merely usually long enough turns into her reporting a failure that did not happen.
     The loop exits the moment the draw goes, so the common case still answers in about a
     second; the ceiling only matters when something is actually wrong, and eleven seconds
     is comfortably inside the thirty the tool call is allowed. #>
  $went = $false
  foreach ($attempt in 1..12) {
    Start-Sleep -Milliseconds 900
    $now = Read-Port $site $chosen.Device.id $index
    if ($now.poe.state -ne 'UP') { $went = $true; break }
  }

  if (-not $went) {
    return "The gateway accepted the request for port $index on $($chosen.Device.name), but the port never " +
           'stopped drawing power, so nothing on it was rebooted. Worth checking on the console.'
  }

  "Power-cycled port $index on $($chosen.Device.name), and watched the power actually drop. It $before. " +
  'Whatever is on it is starting again - give it a minute before asking whether it is back.'
}

<#
  Switching one port's PoE off, and on again, and leaving it that way.

  **This is the other API.** The integration API cannot do it - its only port action is
  `POWER_CYCLE` - so this reaches the older `/proxy/network/api` that the UniFi web UI itself
  uses, and edits `port_overrides[].poe_mode` on the device: `off` to cut it, `auto` to give
  it back. That is a *configuration* change and it survives a reboot, which is the point and
  also the reason it is worth being careful with.

  **It authenticates with the API key, not the account.** The key reaches these endpoints and
  is allowed to write; the `Octavia` account is deliberately read-only and a `PUT` under it
  comes back `api.err.NoPermission`. So this needs no password, and the read-only account
  stays read-only.

  Two things it does not pretend about. `auto` is UniFi's default, not necessarily what the
  port was on before - the previous mode is named in the reply so an unusual one is visible
  rather than silently flattened. And a port with no override row is left alone rather than
  having one invented, because a row this did not write is a row it does not know how to
  put back.
#>
function Set-PortPower([string]$query, $port, $on) {
  if ($null -eq $port) { return 'Which port? The port number is required.' }
  if ($null -eq $on) { return 'On, or off? That has to be said.' }

  $index = 0
  if (-not [int]::TryParse("$port", [ref]$index)) { return "'$port' is not a port number." }

  $wanted = [bool]$on
  $mode = if ($wanted) { 'auto' } else { 'off' }

  $site = Get-SiteId
  $chosen = Resolve-PoeDevice $query $site
  if ($chosen -is [string]) { return $chosen }

  $target = $chosen.Detail.interfaces.ports | Where-Object { $_.idx -eq $index }
  if (-not $target) { return "$($chosen.Device.name) has no port $index." }
  if (-not $target.poe) {
    return "Port $index on $($chosen.Device.name) does not supply PoE, so there is nothing to switch."
  }

  if ([bool]$target.poe.enabled -eq $wanted) {
    $already = if ($wanted) { 'already on' } else { 'already off' }
    return "PoE on port $index of $($chosen.Device.name) is $already. Nothing to do."
  }

  $legacy = Get-LegacyDevice $chosen.Device
  if ($legacy -is [string]) { return $legacy }

  $rows = @($legacy.Device.port_overrides)
  $row = $rows | Where-Object { $_.port_idx -eq $index }
  if (-not $row) {
    return "Port $index on $($chosen.Device.name) has no port configuration to edit, so its PoE cannot be " +
           'switched from here. It can be changed on the UniFi console.'
  }

  $was = if ($row.poe_mode) { $row.poe_mode } else { 'unset' }
  if ($row.PSObject.Properties.Name -contains 'poe_mode') { $row.poe_mode = $mode }
  else { $row | Add-Member -NotePropertyName poe_mode -NotePropertyValue $mode }

  $body = @{ port_overrides = $rows } | ConvertTo-Json -Depth 12
  $result = Invoke-RestMethod -Method Put -SkipCertificateCheck -TimeoutSec 30 `
    -Uri "https://$unifiHost/proxy/network/api/s/$($legacy.Site)/rest/device/$($legacy.Device._id)" `
    -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
    -ContentType 'application/json' -Body $body

  if ($result.meta.rc -ne 'ok') {
    $why = if ($result.meta.msg) { ": $($result.meta.msg)" } else { '' }
    return "The gateway refused to change port $index$why."
  }

  # Same reason as the cycle: accepted is not done. The setting lands within a few seconds.
  $seen = $null
  foreach ($attempt in 1..8) {
    Start-Sleep -Milliseconds 900
    $seen = Read-Port $site $chosen.Device.id $index
    if ([bool]$seen.poe.enabled -eq $wanted) { break }
  }

  if ([bool]$seen.poe.enabled -ne $wanted) {
    $reads = if ($seen.poe.enabled) { 'on' } else { 'off' }
    return "The gateway accepted the change to port $index on $($chosen.Device.name), but the port still " +
           "reads $reads. Worth checking on the console."
  }

  $note = if ($wanted -and $was -ne 'auto' -and $was -ne 'off') { " It was set to '$was' before, not 'auto'." } else { '' }
  $tail = if ($wanted) { 'it has power again' } else { 'it stays off until it is switched back on' }
  $now = if ($wanted) { 'on' } else { 'off' }

  "PoE on port $index of $($chosen.Device.name) is now $now, and $tail.$note"
}

<#
  The firewall, read and never written.

  **Reading it is almost all of the value and none of the risk**, which is why it is the
  whole of this tool. A wrong answer about a rule costs a sentence; a wrong *change* to one
  can take the house off the internet from another room, and there is no undo she can offer.

  **66 policies and 35 of them enabled, and listing them plainly is useless.** They are the
  default matrix between six zones, so the names repeat - "Allow All Traffic" appears nine
  times, "Block Invalid Traffic" six - and a flat list is thirty-five lines that say nothing.
  What a person means by "what is my firewall doing" is the *zone pairs*: Internal to
  External allows everything, Hotspot to Internal blocks it. So that is what it answers.

  **Anything not SYSTEM_DEFINED is called out first**, because on this gateway nothing is,
  and the first rule somebody adds by hand is the one they will later ask about.
#>
function Get-Firewall([string]$query) {
  $site = Get-SiteId

  $zones = @{}
  foreach ($z in (Invoke-Unifi "sites/$site/firewall/zones?limit=50").data) { $zones[$z.id] = $z.name }

  $all = (Invoke-Unifi "sites/$site/firewall/policies?limit=200").data
  if (-not $all) { return 'The gateway returned no firewall policies at all.' }

  $named = $all | ForEach-Object {
    $from = if ($_.source.zoneId -and $zones[$_.source.zoneId]) { $zones[$_.source.zoneId] } else { 'anywhere' }
    $to = if ($_.destination.zoneId -and $zones[$_.destination.zoneId]) { $zones[$_.destination.zoneId] } else { 'anywhere' }
    [pscustomobject]@{
      Name = $_.name; From = $from; To = $to
      Action = $_.action.type; Enabled = [bool]$_.enabled
      Logging = [bool]$_.loggingEnabled
      Index = [int64]$_.index
      Custom = ($_.metadata.origin -ne 'SYSTEM_DEFINED')
    }
  }

  <# **Every word has to match something, rather than the whole phrase matching one thing.**

     The obvious `-like "*$query*"` reads fine and fails on the only question anyone actually
     asks: *"what does the firewall do between the hotspot and my internal network"* arrives
     as `Hotspot Internal`, which is not a substring of any rule name or either zone, so it
     answered "no firewall rule matches". A model handed *no rules* concluded there were none
     and said the Hotspot could reach the Internal network - the exact opposite of the truth,
     confidently. **A search that cannot express "between these two" is worse than no search**,
     because its empty answer is indistinguishable from an empty firewall. #>
  if ($query) {
    $words = @($query -split '[^A-Za-z0-9_-]+' | Where-Object { $_.Length -gt 0 })

    $hits = @($named | Where-Object {
      $rule = $_
      # Every word must appear somewhere in the rule. "Hotspot Internal" therefore means
      # rules touching both zones, and "block hotspot" means blocking rules touching one.
      @($words | Where-Object {
        $rule.Name -like "*$_*" -or $rule.From -like "*$_*" -or $rule.To -like "*$_*" -or
        $rule.Action -like "*$_*"
      }).Count -eq $words.Count
    })

    if ($hits.Count -eq 0) {
      return "No firewall rule mentions all of '$query'. The zones are: " +
             "$((($zones.Values | Sort-Object) -join ', ')). Rules are named things like " +
             "'Allow All Traffic' and 'Block Invalid Traffic'."
    }

    $lines = @("$($hits.Count) firewall rule(s) matching '$query':")
    foreach ($r in ($hits | Sort-Object From, To, Name)) {
      $state = if ($r.Enabled) { 'on' } else { 'off' }
      $lines += "  $($r.From) to $($r.To): $($r.Action.ToLowerInvariant()) - $($r.Name) ($state)"
    }
    return $lines -join "`n"
  }

  $on = @($named | Where-Object { $_.Enabled })
  $custom = @($named | Where-Object { $_.Custom })

  $lines = @("$($all.Count) firewall rules, $($on.Count) of them switched on, across these zones: " +
             "$((($zones.Values | Sort-Object) -join ', ')).")
  $lines += ''
  $lines += 'What the enabled rules do, by zone:'

  <# Grouped, because the interesting fact is "Internal to External allows everything" and
     the nine rules adding up to it are not.

     **The last rule by index is the answer**, not the set of actions present. UniFi walks
     them in order and the catch-all at the end decides anything the specific rules did not
     match, so a pair with both an allow and a block reported as "allow and block" says
     nothing at all - it is true of nearly every pair here. The catch-all is named, and the
     rules sitting in front of it are counted rather than listed. #>
  foreach ($g in ($on | Group-Object { "$($_.From) to $($_.To)" } | Sort-Object Name)) {
    $ordered = @($g.Group | Sort-Object Index)
    $last = $ordered[-1]
    $before = $ordered.Count - 1

    $tail = if ($before -eq 0) { '' }
            elseif ($before -eq 1) { ', with 1 more specific rule before it' }
            else { ", with $before more specific rules before it" }

    $lines += "  $($g.Name): $($last.Action.ToLowerInvariant()) by default ($($last.Name))$tail"
  }

  $lines += ''
  if ($custom.Count -eq 0) {
    $lines += 'Every one of them is a UniFi default. Nobody has added a rule of their own here.'
  } else {
    $lines += "$($custom.Count) rule(s) were added by hand rather than by UniFi:"
    foreach ($r in $custom) {
      $state = if ($r.Enabled) { 'on' } else { 'off' }
      $lines += "  $($r.From) to $($r.To): $($r.Action.ToLowerInvariant()) - $($r.Name) ($state)"
    }
  }

  $logging = @($named | Where-Object { $_.Logging })
  if ($logging.Count -eq 0) {
    $lines += 'None of them log what they block, so the security log will not say which rule matched.'
  }

  $lines += 'This reads the firewall and cannot change it.'
  $lines -join "`n"
}

<#
  Any network device by name, for the tools that are not about PoE.

  `Resolve-PoeDevice` answers a narrower question and is kept separate on purpose: a restart
  is meaningful for an access point that supplies no power at all, and folding the two would
  have this refuse the most likely thing anyone asks it to reboot.
#>
function Resolve-Device([string]$query) {
  $site = Get-SiteId
  $devices = (Invoke-Unifi "sites/$site/devices?limit=50").data
  if (-not $devices) { return 'The gateway lists no devices.' }

  if (-not $query) {
    if ($devices.Count -eq 1) { return $devices[0] }
    return "Which one? There is: $((($devices.name) -join ', '))."
  }

  $hits = @($devices | Where-Object { $_.name -like "*$query*" -or $_.model -like "*$query*" })
  if ($hits.Count -eq 0) { return "There is no network device matching '$query'. There is: $((($devices.name) -join ', '))." }
  if ($hits.Count -gt 1) { return "More than one device matches '$query': $((($hits.name) -join ', ')). Say which." }
  $hits[0]
}

<#
  Restarting a piece of network hardware.

  `RESTART` is the only action a *device* accepts, established the same way the port's was -
  by sending an invalid one and reading the refusal.

  **Restarting the gateway is not like restarting an access point**, and the difference is
  worth a sentence rather than a shrug: the UDM is the router, the switch, the DNS server and
  the thing this tool is talking through. Rebooting it takes the whole network down for
  several minutes, this server's own connection with it, and any call she is in the middle of.
  So the gateway is named explicitly in the answer, and a person confirming a reboot is told
  which of the two they are getting.

  Nothing is verified afterwards, and that is a deliberate exception to the rule the two PoE
  tools follow. A device that is restarting is unreachable for minutes; polling for it would
  hold the tool call open long past its timeout and prove only that a reboot is slow. What is
  checked is that the request was *accepted* - and the wording says exactly that much, rather
  than claiming the reboot has finished.
#>
function Restart-Device([string]$query) {
  $site = Get-SiteId
  $chosen = Resolve-Device $query
  if ($chosen -is [string]) { return $chosen }

  $isGateway = $chosen.model -like '*Dream Machine*' -or $chosen.model -like '*UDM*' -or $chosen.model -like '*Gateway*'

  try {
    Invoke-RestMethod -Uri "$base/sites/$site/devices/$($chosen.id)/actions" `
      -Method Post -SkipCertificateCheck -TimeoutSec 30 `
      -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
      -ContentType 'application/json' -Body '{"action":"RESTART"}' | Out-Null
  }
  catch {
    $code = $_.Exception.Response.StatusCode.value__
    return "The gateway refused to restart $($chosen.name) (HTTP $code). It is still running."
  }

  if ($isGateway) {
    "$($chosen.name) has been told to restart. It is the gateway, so the whole network - " +
    'including this connection - goes down with it for a few minutes. Nothing else will answer until it is back.'
  } else {
    "$($chosen.name) has been told to restart. It takes a couple of minutes to come back, and " +
    'anything connected through it is offline until then.'
  }
}

<#
  Letting one client onto the network, or taking it off.

  `AUTHORIZE_GUEST_ACCESS` and `UNAUTHORIZE_GUEST_ACCESS` are the only two actions a *client*
  accepts. They move `access.type` between `DEFAULT` and the blocked state, which is what is
  read back afterwards - the answer says what the gateway reports, not what was asked for.

  **The client is found the way a person names one**, by part of a name, an address or a MAC,
  because nobody says "unauthorise 4efb4cde-b0ae-3e7d-beef-d71091482222". An ambiguous name
  lists the matches rather than picking, for the same reason the PoE tools do: guessing wrong
  cuts off the wrong person's device.
#>
function Set-ClientAccess([string]$query, $on) {
  if (-not $query) { return 'Which device? Give a name, an address or a MAC.' }
  if ($null -eq $on) { return 'On, or off? That has to be said.' }

  $wanted = [bool]$on
  $site = Get-SiteId
  $clients = (Invoke-Unifi "sites/$site/clients?limit=200").data

  $hits = @($clients | Where-Object {
    $_.name -like "*$query*" -or $_.ipAddress -like "*$query*" -or $_.macAddress -like "*$query*"
  })

  if ($hits.Count -eq 0) { return "Nothing connected matches '$query'." }
  if ($hits.Count -gt 1) {
    return "More than one thing matches '$query': $((($hits.name) -join ', ')). Say which."
  }

  $client = $hits[0]
  $action = if ($wanted) { 'AUTHORIZE_GUEST_ACCESS' } else { 'UNAUTHORIZE_GUEST_ACCESS' }

  try {
    Invoke-RestMethod -Uri "$base/sites/$site/clients/$($client.id)/actions" `
      -Method Post -SkipCertificateCheck -TimeoutSec 30 `
      -Headers @{ 'X-API-KEY' = $apiKey; 'Accept' = 'application/json' } `
      -ContentType 'application/json' -Body "{`"action`":`"$action`"}" | Out-Null
  }
  catch {
    $code = $_.Exception.Response.StatusCode.value__
    $why = if ($code -eq 400) {
      ' The gateway would only do this for a client on a guest network, and this one is not on it.'
    } else { '' }
    return "The gateway refused to change access for $($client.name) (HTTP $code).$why It is unchanged."
  }

  # Accepted is not done, the same as everywhere else here.
  $seen = $null
  foreach ($attempt in 1..8) {
    Start-Sleep -Milliseconds 900
    $seen = ((Invoke-Unifi "sites/$site/clients?limit=200").data | Where-Object { $_.id -eq $client.id })
    if (-not $seen) { break }
    if ($wanted -and $seen.access.type -eq 'DEFAULT') { break }
    if (-not $wanted -and $seen.access.type -ne 'DEFAULT') { break }
  }

  $now = if (-not $seen) { 'gone from the network entirely' } else { "reported as '$($seen.access.type)'" }
  $did = if ($wanted) { 'allowed onto the network' } else { 'taken off the network' }

  "$($client.name) at $($client.ipAddress) has been $did, and the gateway now has it $now."
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
    annotations = @{ readOnlyHint = $true }
    description = 'List the UniFi network hardware - gateway and access points - with model, state, address and firmware version.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'list_clients'
    annotations = @{ readOnlyHint = $true }
    description = 'List everything currently connected to the network, wired and wireless, with name, address and how long it has been connected. This is how to answer who or what is at home.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'get_status'
    annotations = @{ readOnlyHint = $true }
    description = 'Read overall network health: gateway load, memory, uptime, current throughput, and how many devices and clients are present.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'find_client'
    annotations = @{ readOnlyHint = $true }
    description = 'Search the connected clients by name, IP address or MAC and report what matches.'
    inputSchema = @{
      type       = 'object'
      properties = @{ query = @{ type = 'string'; description = 'Name, address or MAC, or part of one' } }
      required   = @('query')
    }
  },
  @{
    name        = 'list_ports'
    annotations = @{ readOnlyHint = $true }
    description = 'Read the switch ports on a network device: whether each one has a link, how fast, and whether it supplies PoE power and is currently doing so. The gateway does not report which client is on which port, so this cannot say what a port feeds.'
    inputSchema = @{
      type       = 'object'
      properties = @{ device = @{ type = 'string'; description = 'Device name or part of one. Omitted means every device.' } }
    }
  },
  @{
    name        = 'power_cycle_port'
    annotations = @{ destructiveHint = $true }
    description = 'Restart the PoE power on one switch port - it goes off and comes back on again by itself, which is how to reboot whatever is plugged into it. That device loses power, and the gateway cannot say what it is. To switch a port off and leave it off, use set_port_power instead.'
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
    name        = 'set_port_power'
    annotations = @{ destructiveHint = $true }
    description = 'Switch the PoE power on one switch port off, or back on again, and leave it that way. Whatever is plugged into that port loses power for as long as it is off, and the gateway cannot say what that is. This changes the port configuration and survives a reboot. To reboot something rather than leave it off, use power_cycle_port.'
    inputSchema = @{
      type       = 'object'
      properties = @{
        port   = @{ type = 'integer'; description = 'Port number, as shown by list_ports' }
        on     = @{ type = 'boolean'; description = 'true to give the port power, false to cut it' }
        device = @{ type = 'string'; description = 'Device name or part of one. Omitted picks the only device that supplies PoE.' }
      }
      required   = @('port', 'on')
    }
  },
  @{
    name        = 'list_firewall_rules'
    annotations = @{ readOnlyHint = $true }
    description = 'Read the firewall: which zones can reach which, what is allowed or blocked between them, and whether anybody has added a rule by hand. Give part of a rule or zone name to see just those. This reads the firewall and cannot change it.'
    inputSchema = @{
      type       = 'object'
      properties = @{ query = @{ type = 'string'; description = 'Part of a rule name or a zone name, such as Internal or Hotspot. Omitted means a summary of all of them.' } }
    }
  },
  @{
    name        = 'restart_device'
    annotations = @{ destructiveHint = $true }
    description = 'Restart a piece of network hardware - an access point, or the gateway itself. Anything connected through it goes offline for a few minutes. Restarting the gateway takes the entire network down, including this connection. This reboots the hardware; to reboot something plugged into a port instead, use power_cycle_port.'
    inputSchema = @{
      type       = 'object'
      properties = @{ device = @{ type = 'string'; description = 'Device name or part of one, as shown by list_devices' } }
      required   = @('device')
    }
  },
  @{
    name        = 'set_client_access'
    annotations = @{ destructiveHint = $true }
    description = 'Allow one connected device onto the network, or take it off. Name it the way a person would - part of its name, its IP address or its MAC. Taking a device off cuts its connection until it is allowed back on. The gateway only does this for clients on a guest network.'
    inputSchema = @{
      type       = 'object'
      properties = @{
        client = @{ type = 'string'; description = 'Name, IP address or MAC, or part of one' }
        on     = @{ type = 'boolean'; description = 'true to allow it onto the network, false to cut it off' }
      }
      required   = @('client', 'on')
    }
  },
  @{
    name        = 'recent_threats'
    annotations = @{ readOnlyHint = $true }
    description = 'Read the security log: intrusion attempts the gateway detected and blocked, grouped by which client set them off. Optionally since a moment in time. This is a read of history and changes nothing.'
    inputSchema = @{
      type       = 'object'
      properties = @{
        since  = @{ type = 'number'; description = 'Unix milliseconds. Omitted means the last hour.' }
        format = @{ type = 'string'; description = "'words' for prose, 'counts' for one name and number per line" }
      }
    }
  },
  @{
    name        = 'list_cameras'
    annotations = @{ readOnlyHint = $true }
    description = 'List the UniFi Protect cameras and whether each one is currently reachable. Use this to say what she can and cannot see.'
    inputSchema = @{ type = 'object'; properties = @{} }
  },
  @{
    name        = 'look_at_camera'
    annotations = @{ readOnlyHint = $true }
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
          'set_port_power' { Set-PortPower $callArgs.device $callArgs.port $callArgs.on }
          'list_firewall_rules' { Get-Firewall $callArgs.query }
          'restart_device' { Restart-Device $callArgs.device }
          'set_client_access' { Set-ClientAccess $callArgs.client $callArgs.on }
          'recent_threats' { Get-Threats $callArgs.format $callArgs.since }
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
