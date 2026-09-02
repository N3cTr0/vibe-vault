---
project: Octavia
tags: [octavia, feature]
---

# Hands — tools and integrations
*Stage 12, seam built in v0.17.0. **The first real server arrived in v0.28.3** — UniFi, five read-only tools, no Home Assistant required — but she **still cannot call one**.* See [[Roadmap]] stage 12.
*Stage 12, seam built in v0.17.0. Configured, connected, listed and tested — **but she cannot call one yet**.* See [[Roadmap]] stage 12.

## What exists and what does not

Read this first, because the gap is easy to misread in either direction.

**Built and tested:** the `IToolProvider` seam, an MCP client speaking stdio JSON-RPC, a registry that namespaces and caches what each server offers, a risk policy, and the confirmation rule for anything irreversible. Configured servers connect at startup, their tools are listed in the log and surfaced in `hello` as `toolServers[]`.

**Not built:** the brain-side loop that lets her actually *call* one. It changes the main conversation path, and it was deliberately not written blind.

So the plumbing is real, reachable and visible. What is missing is the last hop.


## The first real server: UniFi *(v0.28.3)*

*The seam had been tested only against `mock-mcp.ps1` since v0.17.0. It now has a real one, and it needed no [[Roadmap|Home Assistant]] at all.*

**The UDM SE answers an official local API of its own.** That is the finding, and it moves the whole stage forward by months: the roadmap's recommendation — *install HA and let it be the only thing she talks to* — is still right about the **house**, because Google Home has no API a Windows program can use. It was wrong about the **network**.

| | |
|---|---|
| UniFi Network Integration v1 | `10.6.101`, at `/proxy/network/integration/v1` — sites, devices, clients, statistics |
| UniFi Protect Integration v1 | `7.2.105`, at `/proxy/protect/integration/v1`, **same key** — cameras and per-camera snapshots |

Both authenticate with an `X-API-KEY` made in the UniFi UI, over the local wire, to a gateway on the same network.

### Five tools, every one a read

`list_devices`, `list_clients`, `get_status`, `find_client`, `list_cameras`.

**`list_clients` is presence**, and that is the one worth noticing. Eighteen named clients — `Kitchen - Plug - Microwave`, `Main Bedroom - Plug - Misc` — so *"is anyone home"* and *"is the freezer still on"* are answerable **before a single smart-home integration exists**. The network already knew.

### Two decisions worth keeping

**Output is written for a model, not a parser.** `UDM SE: online, up 13d 1h` rather than the raw JSON, which spends most of its tokens on identifiers nothing downstream reads. Failures come back as text for the reason the seam already required: a model told *"the gateway could not be reached"* can say so out loud, where an exception only ends the turn.

**Network and Protect are one server rather than two**, which departs from this note's own *independently broken-able* principle above. Two applications, one appliance, one address, one key — when the UDM is unreachable both are, so there is no independence to preserve and a second process would buy nothing. If Protect ever moves to its own NVR, splitting it is a copy and a config entry.

### What is honestly not there

**Both cameras are offline.** `Front Door` and `Back Garden`, both `DISCONNECTED`, and a snapshot returns `503` rather than an image — so the state is real rather than a stale flag. `list_cameras` reports what is **reachable** rather than what exists, because *"she has cameras"* and *"she can see anything"* are separate claims and only one of them is currently true.

**The snapshot tool is not built.** Protect returns a JPEG and MCP can carry an image, but the brain-side loop does not exist *and* neither camera is online. Two blockers, and writing against neither would be guessing — the mistake this project's notes have recorded three times. See [[Lessons Learned]].

### The check that matters

`EarsTest -- unifi` drives the real gateway and **skips** when no key is configured, so it stays green on a machine with no house — the same bargain the model and microphone probes make.

Eight assertions, and the one worth having is **"every tool is judged a read"**. `RiskOf` looks for its dangerous words first, so a description that gained a `restart`, a `reset` or an `order` would quietly turn a network status query into something she stops to ask permission for — and *nothing else in the system would notice*. The safety of these tools is a property of their **wording**, so the wording is what is asserted. Broken on purpose to watch it go red.

> **She still cannot call any of them.** The last hop is unchanged and is now the only thing between her and a network she can answer questions about. What changed is that it finally has something real, local and entirely harmless to be written against — which was the whole argument for doing UniFi before the house.
## Why MCP rather than N integrations

The argument that decided it: MCP is a published protocol with a tool-definition shape both Claude and local models can be handed, it keeps each integration **out of process** behind a boundary, and it means a new capability is a new *server* rather than a new branch inside `OctaviaSession`.

That is the same bet as `IFaceTransport` and `IBrain`, made a third time — and it is the only version of this that survives contact with a house. Home Assistant, UniFi and whatever comes after are independently useful and independently broken-able; one of them being down must not be a code path in the thing that talks to you.

## The risk policy

`ToolRisk` is a property of the **tool**, not of the sentence that reached it. That distinction is the whole design:

| Risk | Means | She may |
|---|---|---|
| `Read` | Reads state, changes nothing | Do it, freely |
| `Act` | Changes something reversible and visible — a light, a scene, a volume | Do it, and say what she did |
| `Confirm` | Awkward or unsafe to undo — locks, alarms, garage doors, anything that deletes | **Never without an explicit yes in the same conversation** |

The reasoning, from `ITool.cs`: *a model that has misheard "turn the lights off" is one token away from having misheard something worse, and the difference between a wrong answer and a dark house is that the second one happened without anybody agreeing to it.*

Gating on the tool rather than on confidence in the transcript is what makes that hold. Confidence is a number that can be high and wrong; a door lock is a door lock every time.

## Naming

Tools are namespaced `<server>__<tool>` — `house__house_get_state`. Two servers offering `get_state` is not a hypothetical, and a collision resolved by load order is a bug that appears once the second integration is added, months after the first.

## What it does when it fails

An unknown tool **answers rather than throws** — "There is no tool called `house__nonsense`". A tool that needs confirming and did not get it answers too: *"That one needs confirming out loud first. Ask, then do it if the answer is yes."* Both are sentences she can say, because the alternative is an exception in a conversation.

## Configuration

`McpServers` in `config.json`, empty by default. Each entry is a command and its arguments; the client spawns it and speaks JSON-RPC over stdio.

`tools\mock-mcp.ps1` is a fake house — a light, a state read and a door lock — which is what the test suite runs against. It exists so the risk policy and the confirmation rule can be exercised without a real house being involved, and it is the reason those paths were tested before anything was plugged in.

## Testing

`tools\EarsTest` covers the whole seam: the server starts, tools are listed and namespaced, schemas survive the round trip, each risk level is judged correctly, a read runs and answers, an unlock is refused unconfirmed and runs once confirmed, and an unknown tool answers rather than throwing.

## Next

Home Assistant is not installed yet; the smart devices are on Google Home. The order that makes sense is **install Home Assistant → point an MCP server at it → then write the tool loop**, so the loop is written against something real. UniFi (a UDM SE at `10.1.1.1`) is the other early candidate: network health and who is home by device presence. See [[Roadmap]] stage 12.
