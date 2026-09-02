---
project: Octavia
tags: [octavia, feature]
---

# Hands — tools and integrations
*Stage 12, seam built in v0.17.0. **The first real server arrived in v0.28.3** — UniFi, five read-only tools, no Home Assistant required — and **v0.29.0 closed the last hop: she calls them.** Through Claude only; the local brain, and so the everyday  profile, cannot yet.* See [[Roadmap]] stage 12.
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

> ~~**She still cannot call any of them.**~~ **She does, as of v0.29.0 — the same day.** Asked *what hardware is on my network right now?* she answered *There is a Dream Machine Pro SE acting as your gateway, and a UniFi AC Pro access point*, and nobody named a tool. The argument for doing UniFi first held exactly: the loop was written and watched running against read-only tools with nothing in the house to break. See the section below.
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

## The last hop: the brain-side tool loop *(v0.29.0)*

*Written the same day as the server it was written against, which was the argument for doing UniFi first.*

> **Q.** What hardware is on my network right now?
> **A.** *Let me check that for you.* You've got two devices online right now. There's a Dream Machine Pro SE acting as your gateway, and a UniFi AC Pro access point. Both are up and running fine.

Nobody named a tool. She decided one was wanted, picked it, called it, and turned the result into speech.

### Four decisions, and what each one cost

**Tool calls are assembled from the stream**, not from a cheap non-streaming probe followed by a streamed answer. That alternative was half the code and would have silently discarded *"Let me check that for you"* — the sentence spoken **before** the call goes out, while the gateway is being queried. She streams so the voice can start early; a design that stops streaming during the slowest part of a turn defeats the reason streaming exists. `content_block_start` opens a call, `input_json_delta` fragments accumulate, nothing is parsed until the block closes.

**The request is byte-identical when there is nothing to offer.** `Ask()` uses two separate object initializers rather than one with a conditional assignment — so *"identical when empty"* is something a reader can check rather than something they trust a serialiser about. The [[The Brain|system-prompt cache breakpoint]] is untouched for anyone with no servers configured.

**Four rounds, then she stops and logs it.** A model answering its own tool result with another call forever costs money every lap and never ends on its own. The ceiling logs when it bites, because a turn that stopped at a limit looks exactly like one that finished.

**The tool exchange is not written to history.** `Conversation` holds strings; widening it to structured blocks would change every brain and the diagnostics bundle. What the tools said is inside the sentence she just spoke, so the next turn keeps the substance and loses only the ability to quote the raw result verbatim.

### The half that is not built, said out loud

`confirmed` is **always false**. A `ToolRisk.Confirm` tool therefore comes back with the registry's refusal, she relays it, and nothing happens. Carrying a spoken *yes* from one turn into the next is its own piece of work and is not done.

Nothing configured today is riskier than a read, so this costs nothing yet — but it is named at the call site in the code, not only here, because **an unbuilt half that looks built is worse than one that fails loudly**, and this is the half where a door gets unlocked.

### How it is proven

`EarsTest -- toolloop` asks two real questions of the real API and the real gateway. That is the only way to know the model *chooses* a tool rather than that the plumbing compiles.

It is deliberately **not** in the default suite: it spends money every run, and *a self-test that spends money is a bad self-test* is already a rule here. Everything up to the last hop stays covered for nothing by the mock and by `EarsTest -- unifi`.

### What is still missing

**Claude only.** `LocalBrain` speaks the OpenAI-compatible API and needs its own `tools` array and `tool_calls` delta handling — a second implementation, and streaming tool-call support varies across Ollama, LM Studio and llama-server.

The `home` profile is a local brain, so **on the everyday profile she still cannot call a tool.** That is the remaining gap, and it is a far smaller one than the one this closed.

### Both brains, the same day *(v0.29.1)*

The section above ended by naming the gap, and the gap was the one that mattered: **the `home` profile is a local brain.** *"She can use tools"* and *"she can use tools on the profile she is actually started under"* were different claims, and only the second is worth anything.

`LocalBrain` now sends an OpenAI-compatible `tools` array and assembles `tool_calls` out of its own stream. qwen2.5:7b, on CPU, answers correctly:

> **Q.** What hardware is on my network right now?
> **A.** Your network has a UniFi Dream Machine PRO SE and an AC Pro access point.

**The index identifies a call across chunks, not the id.** Ollama returns a streamed tool call *whole* — one chunk, id and name and arguments all complete — so the naive reading works perfectly here and would break on llama-server or LM Studio, which may fragment the arguments across many chunks and send the id only in the first. Every field accumulates: free when there is one fragment, correct when there are twenty. See [[Lessons Learned]] — *write against the format, not against the one server that happens to be running*.

**A machine with no tool servers must not start sending a `tools` key.** Some servers refuse it outright when the model has no tool template, so *identical when there is nothing to offer* is a stricter requirement here than for the hosted brain. Met the same way: two request shapes, not one with a conditional key.

**A 7B model embellishes.** Asked about the hardware it added *"with their firmware versions up to date"*, which no tool said and nothing checked. The loop was working perfectly; the embellishment is downstream of it. **The reliability of a tool call is not the reliability of the sentence built from it** — which is an argument for keeping anything dangerous behind confirmation no matter how good the plumbing gets, and the reason `confirmed` staying false costs so little today.

## She can see through the house's cameras *(v0.31.0)*

*`look_at_camera` fetches a UniFi Protect snapshot and hands it to the model as a picture. Both blockers named in the v0.28.3 note are gone: the loop landed in v0.29.0, and the owner switched a camera on.*

> **Q.** Have a look outside and tell me what you can see.
> **A.** That's odd, the Back Garden camera seems to have been moved or knocked, it's actually pointing at the inside of a room right now, showing a ceiling fan and a wardrobe. And the Front Door camera is offline, so I can't check that one at all.

The frame confirms it exactly — a lit ceiling fan, a wooden wardrobe, exposed roof beams. **She described what was there rather than what the camera was called**, and flagged the mismatch unprompted. That is the whole argument for handing a model the picture instead of a description of one, and it arrived on the first attempt.

### The seam was text-only, and is not any more

`IToolProvider.CallAsync` returned a `string`, and [[Architecture|McpClient]] carried a comment saying an image block *"would need the vision path and is left for whenever that matters"*. A camera is when it matters: the useful answer to *what is at the gate* is not a sentence about a JPEG.

`ToolAnswer` carries text and an optional image, with an **implicit conversion from `string`** so every text-only path in the codebase reads exactly as it did.

Three rules came with it, each earning its place:

- **An image-bearing answer must also say in words what it captured.** Not a nicety — [[The Brain|LocalBrain]] has no eyes, takes the text and logs that it dropped a picture, so a local turn stays usable instead of blank.
- **Only the first image is kept.** A tool returning twelve frames would otherwise put twelve full images into one turn, which is a bill and a context window rather than an answer.
- **The picture goes inside the `tool_result`**, not loose in the conversation, so *"this is what the camera saw when you asked"* stays attached to the asking. A frame floating in the history becomes a photograph with no caption the moment a second one arrives.

### Two refusals, because they are two problems

A camera nobody has heard of is a **typo**, and the tool lists the real names. A camera that exists but is unreachable is a **fact about the house**. Saying *"not reachable"* for both would send somebody looking for a name that was never wrong.

`highQuality` is asked for and not required: this G5 Bullet answers *"Camera does not support full HD snapshot"* with a 400, so the standard frame is fetched instead — written that way because the next camera may well support it.

### What the local brain did

Declined honestly rather than bluffing: *"I can't look outside yet, but I can check the cameras. Which camera would you like me to check?"* Not ideal — it had a tool it could have called — but not a lie either, which is the failure mode that would have mattered.

### Proven

`EarsTest -- unifi` gained three assertions and **adapts to the day**: with a camera online there must be image bytes and the words must name that camera; with none online it says the snapshot went untested rather than failing. It finds the camera by parsing `list_cameras`' own prose, so if that wording stops being readable the check notices too.

## A spoken yes, and how narrowly it counts *(v0.32.0)*

*The last thing between her and a tool that does something. The confirmation rule has existed since v0.17.0 and had never been reachable: every call went out with `confirmed: false`, so a dangerous tool could only ever be refused.*

```
> Unlock the front door.
  Do you want to unlock the front door?
> yes
  The front door is now unlocked.
```

The log confirms what the words claim — `needs confirmation; not run`, then `confirmed by the last thing said`, then the call. Answering **no** produces the refusal and nothing else.

### The rule

**Consent lasts exactly one turn.** It is read and cleared at the top of every turn, so a question she asked is answerable by the very next thing said and by nothing after it. A consent that survived several turns would let a *"yes"* about something else entirely open a door — and *"say yes to the next thing she asks"* is a sentence a television can produce. See [[The Attention Gate]] for why that is not a hypothetical here.

**Consent is to a call, not to a tool.** The arguments are part of it: *"yes"* about the back door is not consent to unlock the front one, however the model phrases its second attempt.

**It refuses when unsure**, because the costs are not symmetric — a yes misread as a no costs one repeated question; a no misread as a yes costs whatever the tool does. Agreement must be present *and* disagreement absent, so *"yes, but not yet"* is not consent.

### One change that is not about the rule at all

The refusal text. Told merely that confirmation is needed, a model will often **ask and answer itself in the same breath** — *"shall I unlock it? Yes, unlocking now"* — which is a confirmation nobody gave. The text now instructs it to ask plainly, in one short question, and say nothing else.

That is worth noticing generally: a permission system's weakest point can be the *wording of its refusal*, not the check.

### Proven twice, both free

`ToolChecks` asserts what the predicate decides — ten cases, every direction that could go wrong. `EarsTest -- confirm` asserts the decision is actually *reached*, driving the whole exchange as a two-turn conversation against the mock house on the local brain, so it costs nothing and unlocks nothing real.
