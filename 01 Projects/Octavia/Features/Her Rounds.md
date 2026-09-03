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

## The first round: the network's security log *(v0.44.0)*

**The design came out of the data, and the plan it replaced was wrong.**

The plan was *"tell him about anything at or above high severity"*. The first eight-hour sample of the real log held **195 events, every one `VERY_HIGH`, at a steady 24 an hour** — and 168 of them came from one machine, which turned out to be the owner's torrent box. Severity is not a filter on that network; it is a constant. A threshold on it would have produced the crying-wolf failure this feature was designed to avoid, arriving by a route nobody had thought of.

> *"I think she should at least learn the system for a week and then start reporting on them, like if this gets installed at a different location my stuff wouldn't apply there."*

So severity is not the signal and volume is not the signal. **Change is** — and what counts as change is learned.

### `Baseline`

**Nothing about any particular network is written down anywhere in this project.** A torrent box on one site and a camera recorder on another are the same thing to it: a name that turns up often.

| | |
|---|---|
| Silent for | `Rounds.LearnForDays`, seven by default |
| Counted from | her **first observation**, not installation — a machine off for three days has three more days to watch |
| Then speaks for | a source never seen while learning, or a known one far outside what it has ever done |
| Survives | a restart. She restarts every time somebody compiles, and a week that began again each time would never end |

**A flagged walk is not learned from.** Folding one in is how a slow escalation teaches her to accept it: each hour a little worse than the last, each hour becoming the new normal, and nothing ever said. A check drives four rising hours and asserts she says all four.

The file is readable on purpose — a person should be able to open it and disagree with what she thinks is normal.

### Two APIs on one appliance

The UniFi **Integration API** the rest of [[Hands]] uses has no event history at all: every event-shaped route 404s, proven with a nonsense-path control. The events are on the **legacy** API behind a cookie login — hence the read-only account and the sealed secrets in [[Conventions & Security Model]].

The log **does not name which rule matched**. She can say who, how many, and what changed; not what it was.

### `rehearseRound`

An hourly errand is a thing nobody sees working until the night it matters. The dev panel's **Rehearse a round** says what a finding sounds like on demand, through the real delivery path.

The sentence says it is a rehearsal, and that is not decoration: it lands in the room's history like any other turn, and a line there claiming a camera was attacked would be read as fact the next time somebody asked.
