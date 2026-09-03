---
project: Octavia
tags: [octavia, feature]
---

# Her Rounds

*The machinery in v0.42.0 (Stage 18). Nothing registered to walk yet.*

**The one thing she does that nobody asked for.**

Everything she had ever said was a reply — to a held button, a typed line, a wake word — and [[Architecture|OctaviaSession]] is built around that: one turn in flight, one room attended, a floor somebody had taken. `Watchman` is the other thing. It walks a route on a clock, and **its normal and correct outcome is that she says nothing at all.**

Asked for on 09/03/2026: *"I would like her to be able to review the logs for suspicious threat activity every hour and let me know if she found something concerning."*

## It reads like a UniFi feature and it is not

The UniFi half is a source of data, and [[Hands]] already knows how to add one. The new thing is that **she begins a turn**, and that silence is the answer almost every hour.

## Four rules, each because the obvious version is worse

| | |
|---|---|
| **It never interrupts** | A finding arriving mid-turn waits up to five minutes and says how long it waited. Cutting across somebody to announce a port scan is worse than the port scan. |
| **It is quiet at night** | 22:30–07:30 by default. Findings inside the window are **held, not dropped** — the point of checking at four in the morning is that somebody hears about it at eight. |
| **Silence is recorded** | Every walk is logged, and Health gained a **Rounds** row: when she last looked, and what came of it. |
| **It never asks the model** | The round's own data decides whether to speak, and composes the sentence. |

That last one is the defence against crying wolf. **A model asked hourly whether anything looks concerning will find something concerning**, and bill for the privilege. The threshold belongs in the data — a severity the source itself reported. The model's job starts *afterwards*, when a person asks a follow-up and the finding is sitting in the room's history for it to answer from.

## The turn she begins

`SayUnprompted` is `RespondTo` entered from the other end, and much smaller, because the brain is not involved.

- It speaks into **the room she last attended**, not the host room. Announcing into an empty study because the study is "hers" would be correct and useless.
- It writes **both halves** into that room's history, with what prompted her in the person's slot. Not a lie about who spoke: a round's finding genuinely is an input arriving from somewhere other than a microphone — and without it, *"what did you find?"* a minute later has nothing to answer from. It also keeps `Conversation` paired, which drops exchanges two at a time and warns that an orphaned assistant turn is rejected outright by some providers.
- It **returns false rather than waiting**, and lets `Watchman` decide how patient to be.

## Two checks that would have failed by the clock

Worth remembering for anything else on a schedule — see [[Lessons Learned]].

- The test config sets **no quiet hours at all** rather than the default overnight ones. A suite run late at night would otherwise have found every *"she says it"* check failing, correctly, for a reason with nothing to do with what was being tested.
- `WalkAsync` takes the time as an argument, because **a quiet window cannot express "always"** — every wrap-around window covers 24 hours minus a minute — so proving a finding is held overnight would have had a one-minute hole in it.

## What is left

The first real round reads UniFi's IPS/IDS feed. The API key reaches **no event history at all** (see [[Hands]]), so it needs a **read-only local UniFi account** — chosen over syslog on 09/03/2026 — put in her config and sealed the way the API key is.
