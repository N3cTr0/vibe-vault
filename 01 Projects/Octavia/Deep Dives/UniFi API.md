---
project: Octavia
tags: [octavia, deep-dive, reference]
---

# UniFi's APIs — what is actually there

*Established by probing the owner's own UDM SE on 09/03/2026, not from documentation. Network application **10.6.102**, Protect **7.2.105**, firmware **5.1.31**. Ubiquiti moves these between releases, so treat versions as part of every claim below.*

**There are three surfaces, and the useful mental model is that they do not overlap.** Two of them are on the box; one is in the cloud.

| Surface | Base | Auth | What it is for |
|---|---|---|---|
| **Integration API v1** | `https://<host>/proxy/network/integration/v1` | `X-API-KEY` header | Inventory and control. Official, stable, documented |
| **Legacy application API** | `https://<host>/proxy/network/api/s/<site>` and `/proxy/network/v2/api/site/<site>` | cookie login + `X-CSRF-Token` | Everything the UI does that the above cannot. Undocumented, moves between releases |
| **Site Manager (cloud)** | `https://api.ui.com/v1/` | `X-API-KEY` | Fleet across consoles. Not used here — she is local-only |

> **Version-specific docs live on the box**: UniFi Network → Settings → Integrations. That page is generated for the version installed, which is worth more than any blog post.

## The single most important fact

**The Integration API has no history of any kind.** It is inventory and control. Every event-shaped route returns 404:

`events`, `alerts`, `threats`, `ips/events`, `logs`, `notifications`, `anomalies`, `statistics`, `firewall`, `wlans`, `portForwards`

**Those are real absences, not a proxy refusing before it routes.** A nonsense path returns a *distinct* 404 naming the path — `No endpoint GET /integration/v1/zzz-nonsense-path` — so a 404 here means the route does not exist. See [[Lessons Learned]] on the earlier version of this mistake, where a `401` was read as proof an endpoint existed.

## Integration API — what does work

| Path (after `/integration/v1`) | Notes |
|---|---|
| `info` | `{"applicationVersion":"10.6.102"}` |
| `sites` | Paged envelope: `offset, limit, count, totalCount, data[]` |
| `sites/{site}/devices` | Add `/{id}` for detail, which includes `interfaces.ports` |
| `sites/{site}/devices/{id}/statistics/latest` | CPU, memory, uptime, uplink rates |
| `sites/{site}/clients` | Wired and wireless. **Presence** is answered from here |
| `sites/{site}/networks` | VLANs |
| `POST sites/{site}/devices/{id}/actions` | Only `RESTART` |
| `POST sites/{site}/devices/{id}/interfaces/ports/{idx}/actions` | Only `POWER_CYCLE` |

**Ask the API what it accepts.** POSTing a deliberately invalid action makes it name the valid set — that is how both action lists above were established, and it cost nothing and cut nothing. There is **no way to switch a PoE port off and leave it off** through this API.

**A wired client carries `uplinkDeviceId` and no port index.** "Port 4 is the front door camera" is not a fact available here, at all.

### Protect, same key, same appliance

`https://<host>/proxy/protect/integration/v1` — `meta/info`, `cameras`, `nvrs`, `viewers`, `lights`, `sensors`, `chimes`, and a JPEG snapshot per camera. Flat arrays, **not** the paged envelope the Network API uses.

`events` and `alarms` are 404 here too.

## Legacy API — the login

```
POST https://<host>/api/auth/login   {"username","password","rememberMe":false}
```

Returns a session cookie and an `X-Updated-CSRF-Token` header. Send that back as `X-CSRF-Token` on everything after. Sessions expire quietly and the next call is a 401 — re-login once and retry.

A **read-only local admin** is enough for everything below. Octavia's account reads `is_super: false`, `is_owner: false`.

### The system log is the event feed

```
POST /proxy/network/v2/api/site/<site>/system-log/<category>   {"pageNumber":0,"pageSize":200}
```

**Valid categories** (anything else 404s — confirmed against a nonsense control):

`all` · `admin-activity` · `update-alert` · `device-alert` · `client-alert` · `critical`

There is **no** `threats`, `security`, `ips`, `vpn`, `firewall` or `triggers` category. Security events arrive under `all` with `category: "SECURITY"`.

A row:

```json
{ "category": "SECURITY", "subcategory": "SECURITY_INTRUSION_PREVENTION",
  "event": "THREAT_BLOCKED", "key": "THREAT_BLOCKED_KNOWN_SOURCE_CLIENT",
  "severity": "VERY_HIGH", "timestamp": 1788427395249,
  "message_raw": "A network intrusion attempt from {SRC_CLIENT} to {DST_IP} has been detected and blocked.",
  "parameters": { "SRC_CLIENT": {...}, "DST_IP": {...}, "DEVICE": {...}, "INITIATOR_ID": {...} } }
```

- `SRC_CLIENT` when the gateway knows the client, otherwise a bare `SRC_IP`. **Both occur**, so a reader must handle either or it will merge unrelated sources.
- **`severity` is not a filter.** Every one of 195 security events in an eight-hour sample read `VERY_HIGH`. See [[Her Rounds]].
- The endpoint has **no `since`**. Page until the timestamps fall behind what you want.

## Dead end: which rule matched

**It is not obtainable.** She can say who, how many and what changed; never *what*. Probed and ruled out:

| | |
|---|---|
| `stat/event`, `stat/alarm`, `stat/ips/event`, `stat/ipsevent` | 404, GET and POST, with and without a window. **Removed in 10.x** — every guide describing them predates this |
| `rest/alarm`, `rest/event`, `rest/ips`, `rest/threat` | 400 |
| `rest/ipsalert` | **200 and empty**, over 12 hours in which the system log held ~290 events. The collection exists and the data no longer lands in it |
| `v2/.../ips/alerts`, `ips/events`, `security/*`, `threat-management`, `insights`, `honeypot` | 404 |
| Resolving `INITIATOR_ID` or a row `id` against seven collections | 404 or 400 |

The signature is on the Threat Management screen in the UI, so something serves it — but nothing reachable under `/proxy/network/`. Worth retrying after a Network upgrade; not worth more probing now.

**Untested caveat:** the account is read-only. A super-admin *might* see more. Escalating a service account's rights to chase a nicety is the wrong trade, so it was not tried.

## Useful odds and ends

- `GET /proxy/network/api/s/default/rest/setting/ips` — `ips_mode` (`ips` = blocking), `enabled_networks`, `enabled_categories`, `dns_filtering`.
- `GET /proxy/network/api/s/default/stat/sysinfo` — `data_retention_days` (90 here), build and version.
- `GET /proxy/network/api/s/default/self` — who the credential is, and whether it is super.
- `GET https://<host>/api/system` — hardware shortname and console name, **no auth needed**.
- The certificate is the UDM's own self-signed one. Verification is skipped, and that is safe *only* because the address is a local gateway on the same wire reached by IP.

## How Octavia uses each

| | |
|---|---|
| `X-API-KEY`, in `McpServers.unifi.Env` | Devices, clients, status, ports, PoE cycling, Protect cameras |
| Read-only account, password DPAPI-sealed via `McpServer.Secrets` | The system log only — see [[Conventions & Security Model]] |

Both live in `tools\unifi-mcp.ps1`. One server rather than two: they are two applications on one appliance at one address, so when it is unreachable both are, and there is no independence to preserve.
