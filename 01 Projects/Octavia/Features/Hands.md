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

## She can change something now *(v0.41.0)*

*"Is it possible to get Octavia to switch off a poe port and back on again?"*

`power_cycle_port` is **the first tool in the project that changes anything**. Until it, every tool she had was a read, which was deliberate — the tool loop was written and watched against something that had nothing in the house to break.

| | |
|---|---|
| `list_ports` | Link, speed, and whether each port supplies PoE and is doing so |
| `power_cycle_port` | Off, then on. A `Confirm` — she asks plainly and stops |

Two facts about the API, both **established by probing rather than assumed**:

- **`POWER_CYCLE` is the only action a port accepts.** Sending a deliberately invalid one made the gateway name the valid set. Off-and-back-on is one atomic action, and leaving a port switched off is not possible through this API at all.
- **The gateway does not say what is on a port.** A wired client carries an `uplinkDeviceId` and no port index. Both the listing and the tool description say so, because a model that infers the mapping from names will be confidently wrong about which camera it is about to reboot.

### The classification is the safety

`RiskOf` guesses risk from the *wording* of a name and description. That was already a hazard in one direction — a read gaining the word "reset" becomes something she stops to ask about. This is the serious direction: **the only thing between "restart the power on port 4" and her doing it unasked is the word "Restart" in that description.** Soften it and the classification drops silently to `Act`, which she may perform on her own. Both directions are now pinned by checks.

> **A check written to test the script found the guard instead.** Calling the tool with a nonsense port expecting the script's refusal returned the *registry's* refusal — the call never reached the script, because a `Confirm` tool does not run without a yes. See [[Lessons Learned]].

Confirmed live against the real gateway: asked to cycle port 4, she said *"Do you want to power cycle port 4 on the UDM?"* and stopped, and the log recorded `needs confirmation; not run`.

## She said she had done it, and she had not *(v0.49.0)*

> *"I asked her to power cycle port 1 on the UDM, she said she did it but the camera that gets its power from it remained operational the whole time."*

**Two independent faults, both producing exactly that sentence.** Either one alone would have been enough, which is part of why it took a session to find: fixing the obvious one would not have fixed the symptom.

### One: the tool never ran

`power_cycle_port` is a `Confirm`, so it needs a spoken yes. The rule that recognises one compared the call's arguments as **raw text** — and the two sides come from two separate generations of a model. The log, once it was asked to say:

```
tool call: unifi__power_cycle_port{"device": "UDM", "port": 1}   ← when she asks
tool call: unifi__power_cycle_port{"port": 1, "device": "UDM"}   ← after the yes
```

Same call. Different key order. Refused.

**127 refusals and one grant in the whole log history** — and the single grant was `house__house_unlock_door` on the mock, which takes no arguments, so there was nothing for the byte comparison to disagree about. The feature described in *[[Hands#A spoken yes, and how narrowly it counts (v0.32.0)|A spoken yes]]* above had essentially never worked outside its own test.

`Conversation.Grants` compares them as JSON now — keys sorted, whitespace ignored, and nothing else relaxed. A different port, a different device, an argument added or dropped is still a different call, because the reason that rule exists is that *"yes"* about the back door is not consent to open the front one.

### Two: it asserted its own success

The tool posted the action, piped the response to `Out-Null`, and returned *"Power-cycled port 1"* without ever looking. It watches the port now — and caught a real failure the first time it ran:

> *"Something's off. The gateway took the request but the power never actually dropped, so whatever's on port 1 didn't reboot. You'll probably need to check it from the UniFi console directly."*

**That is the sentence the original report should have produced.** See [[Lessons Learned]].

### Why the checks were green throughout

Every consent check handed the **same C# string** to both sides of the comparison:

```csharp
talk.AwaitYes(door, back);
Conversation.Grants(Asked().TakeConsent(), door, back, "yes");
```

A byte-for-byte match cannot fail against itself. The checks were not weak about the rule; they were **testing a situation that never occurs**, and the situation that does occur — two generations of the same call — was the one thing they could not express. They now differ as text and agree as meaning.

## Switching a port off, and leaving it off *(v0.49.0)*

> *"I want to be able to ask her to switch off PoE on certain ports and switch them on again."*

`set_port_power` takes a `port` and an `on`. **It will not guess the direction**: a missing `on` is a question, not a default, because one default reboots a camera and the other leaves it dark.

| `power_cycle_port` | Off and back on by itself. For rebooting something |
|---|---|
| `set_port_power` | Off, or on, and it stays there. For leaving something without power |

The two descriptions point at each other, and a check pins that they do — they are one word apart in English and very different in effect.

### It is on the other API

The v0.41.0 note above says leaving a port off *"is not possible through this API at all"*, and that was true and remains true — re-established this release by asking for seven more actions and reading every refusal, all of which name the valid set as exactly `POWER_CYCLE`.

What was wrong was the unstated leap from *this API cannot* to *it cannot be done*. The older `/proxy/network/api` — the one the UniFi web UI itself calls — holds `port_overrides[].poe_mode`, and setting it to `off` or `auto` does exactly what was asked. It is a **configuration** change, so it survives a reboot, which is the point and also the reason to be careful with it.

Two things it does not pretend about: `auto` is UniFi's default rather than necessarily what the port was on before, so an unusual previous mode is named in the reply instead of being silently flattened; and a port with no override row is left alone rather than having one invented, because a row it did not write is a row it does not know how to put back.

`power_cycle_port` now refuses a port whose PoE is switched off, and one that is on but has nothing drawing from it — the gateway answers that second case with a bare `422`.

## One credential instead of two *(v0.49.0)*

The security log lived behind a username and password, on the stated belief that **the API key could not reach the legacy API**. Half of that was true: it *accepts* a cookie session. The untested half was wrong — it accepts `X-API-KEY` just as readily, for reads and for writes.

So a real UniFi **account password** — worth considerably more to an attacker than the key is — sat in the secret store buying nothing. The login, the session, the CSRF token and the 401 retry that existed only to renew them are gone. `UNIFI_USERNAME` and `UNIFI_PASSWORD` are read by nothing; the stored password and the read-only account can both be deleted.

**A wrong conclusion drawn from a real experiment.** Every *integration* route genuinely 404s, and that was measured carefully with a nonsense-path control. The finding was then extended one step past what it measured. See [[Lessons Learned]].

## Risk is declared, not guessed *(v0.49.0)*

`RiskOf` reads a tool's prose for dangerous words, and everything above about the wording being the safety was true. `set_port_power` came out `Confirm` — but **only because a sentence pointing at the other tool contained the word "reboot"**. Tidying that line would have dropped a tool that cuts power to `Act`, which she performs unasked, and nothing would have noticed.

Every tool in the UniFi server now carries MCP's standard `annotations`: `readOnlyHint` on the eight reads, `destructiveHint` on the two writes. The heuristic still runs underneath for servers that annotate nothing, and it is only ever allowed to be *more* careful — a declared `destructiveHint: false` is ignored outright, so a third-party server cannot talk its way out of a question.

A check asserts the annotation is what makes the difference, by confirming the heuristic alone says `Act`.

## Every call and every answer is in the log *(v0.49.0)*

The question this release answers — *did the tool run, and what did it say* — could not be answered from the log at all. It recorded that a tool was called and nothing about how it went.

| `debug` | `tool call: unifi__power_cycle_port{"port": 1}` — the arguments as sent |
| `debug` | `tool answer: ... -> ` the reply, folded to one line and truncated |
| `info` | every consent decision, **with its reason**: wrong tool, different arguments, read as a refusal, or neither a yes nor a no |

Both brains, the same lines. The consent reasons are the ones that matter: a refusal used to be entirely silent, which is how a rule could fail 127 times without anybody being able to see it.

## The same bug from the other side *(v0.49.1)*

> *"I asked her to switch off the PoE to port 1 and she said she can't do it."*

Reported within the hour of v0.49.0 shipping, and **v0.49.0 is why it took one run to find**. The consent logging added there named the reason immediately:

```
consent was for set_port_power{"port":1,"on":false},
not             set_port_power{"device":"unifi-gateway","port":1,"on":false}
```

The model **added an argument** after the yes. v0.49.0 taught the comparison to ignore key order and whitespace — correct for the half it addressed, and structurally incapable of addressing this half.

### The design was the bug, not the comparison

**It asked two independent generations of a model to agree on a JSON object.** They agree often enough to look fine and not often enough to work, and every conceivable comparison rule sits somewhere on a spectrum between *"loops for ever"* and *"consents to something nobody agreed to"*. There is no correct point on that spectrum, which is the sign that the spectrum is wrong.

So the model is no longer asked to reproduce anything. `Conversation.Authorises` returns the call **she described out loud**, and that is what runs.

That is not a relaxation — it is strictly safer. The front-door-after-a-back-door-yes case used to be *caught* by the comparison; now the front door is never what executes. And what runs is, by construction, the call the person actually heard described.

A yes authorises **one** call, cleared on use, so it is never a standing permission for the rest of the turn.

### The checks had the same shape of hole, one level up

Every consent check asserted the **comparison**, and the comparison was never the weak part. What went untested was the claim the comparison rested on: *that a model asked the same thing twice writes it the same way*. Nothing in the file could have expressed that, because both sides were written by the same hand.

They now assert what a yes **returns** rather than what it compares. See [[Lessons Learned]].

> **Twice in one day, at two different levels.** v0.49.0's checks shared a string between both sides of a comparison; v0.49.1's tested a comparison whose premise was false. Both are the same error — *testing the mechanism instead of the assumption it rests on*.

## She hears you, in the log *(v0.49.1)*

`turn in room 'host'` recorded that a turn had happened and **nothing about what it was**, which makes the commonest report in this project unanswerable from the log. The same sentence typed into a probe reached the tool; spoken through `small.en` it did not, and there was no way to tell which of the three layers — transcript, model, tool — had gone wrong.

```
heard in 'host': switch off the poe on port 1
said in 'host': PoE on port 1 has been switched off and will stay that way...
```

At `debug`, because it is a transcript of somebody's room and the default level should not collect one.

### `ask` — a question without a microphone

```
dotnet run --project tools/EarsTest -- ask home "switch off the poe on port 1" yes
```

**The profile is the point.** `home` is a local brain, so a tool that works on Claude may simply never be reached in the room — *"she can use a tool"* and *"she can use a tool on the profile she is actually started under"* are different claims, and only the second is worth anything. It is the second that was broken.

`qwen2.5:7b-cpu`, after the fix:

> *"PoE on port 1 has been switched off and will stay that way until it is turned back on."*
