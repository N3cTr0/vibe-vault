---
project: Octavia
tags: [octavia, feature]
---

# The Attention Gate

*Stage 9, v0.9.0.* What she answers, and what she lets go.

## The problem it exists for

A microphone in a room hears the television, half a phone call, someone singing along, and both sides of a conversation she is not in. Every one of those reaches the recogniser as a clean sentence.

Two things go wrong without a gate, and they are different kinds of bad:

- **She answers all of it.** Insufferable, and the reason nobody leaves an assistant listening.
- **You pay for all of it.** Every stray line becomes a Claude call.

This is the component [[Choosing a Local Model]] was really about. Stage 2 said the local model would stay after development because "Stage 9 needs a cheap gate in front of the expensive model" — this is that gate, and the interface it needed already existed.

## Two layers, cheapest first

**The free layer** settles most traffic in microseconds, with no model at all:

| Rule | Verdict | Why |
|---|---|---|
| Contains one of `WakeNames` | **answer** | The one unambiguous signal there is |
| Within `GateFollowUpSeconds` of her last answer | **answer** | "And what about tomorrow?" is addressed by context |
| Shorter than twelve characters | ignore | Too short to carry an address or a request |

The follow-up window is not a nicety. Without it her name would have to be said every single turn, which nobody does and nobody would tolerate.

**The model layer** takes what is left — one call, to the *small local* model, which is free. **No paid model is ever used to decide whether to use a paid model.**

## Two properties that matter more than accuracy

**It fails open.** If the gate times out or the server is down, she answers. A companion who goes silent because a helper model died is *broken*; one who occasionally replies to the television is merely annoying. The log records which happened, so failing open is never invisible.

**It is never silent.** A declined line goes to the log *and* to the face as an `overheard` message carrying the reason, shown faintly in the transcript. This is the [[Diagnostics]] principle applied to behaviour: "she ignored me" has to be a question with an answer.

And **typed input is never gated** at all. If you took the trouble to type it, you meant it, and a gate that second-guesses a deliberate act is only ever wrong.

## What it measures

`EarsTest -- gate` scores eighteen labelled lines drawn from what a room microphone actually picks up. On this VM:

> 14 agreed · **1 ignored-you** · 3 answered-noise · median **1.2 s**

The score is the least interesting number there. The two errors are not equal:

- a **false no** is Octavia ignoring you, which is what makes her feel broken
- a **false yes** costs one model call

So the tuning target is the first column, not the total. The probe prints that asymmetry above the table so nobody optimises the wrong one, and warns against fitting a prompt to eighteen lines.

## The instruction, and the word that broke it

The first version asked whether a line was *"addressed to an assistant"*. That sounds exactly right and is wrong: the model read **addressed** as **named**, and threw away "tell me a short joke" and "can you turn the volume down a bit". People do not say her name every time.

Asking instead whether it is *a question or a request that an assistant should answer* — and saying explicitly "even with no name in them" — took false noes from three to one.

Halving the instruction's length also halved the median latency. On a modest machine the prompt dominates: every token is processed on every call.

## Two findings that changed the design

**The gate model must be the same model as the brain.** A smaller separate model sounds obviously better and is not: two models cannot both stay resident, so the server evicts one to load the other on every utterance. Measured at **24 seconds** against **0.7** for a warm call. The self-test fails loudly when they differ, because nothing about that misconfiguration is visible from the outside.

**A reasoning model is useless as a gate.** `qwen3:1.7b` spent its entire 24-token budget deliberating a yes-or-no question and returned an empty answer. There is no portable way to stop it — `think`, `/no_think` in the prompt, and `chat_template_kwargs.enable_thinking` were each tried against Ollama's OpenAI-compatible endpoint and each was ignored. Only Ollama's *native* API honoured it, which would have abandoned the "any OpenAI-compatible server" contract that [[The Brain]] deliberately holds.

## Where the latency actually lands

> **Every number here was measured on three cores with no GPU.** Most of the 1.2 s is prompt processing, which is exactly what better hardware fixes. Re-measure with `EarsTest -- gate` on the new machine before drawing any conclusion from these figures. See [[Moving To The New Machine]].

1.2 s sounds unaffordable for something added to every utterance — until you notice what reaches the model layer. Anything **named** and anything in the **follow-up window** is settled for free. So the model call falls mostly on lines she is about to *ignore*, where nobody is waiting for an answer.

The number that hurts is the delay before a genuine cold question with no name in it. That is the median, not the worst case, and it is why the timeout is generous rather than tight: a tight one bought nothing but timeouts, each of which then failed open and paid for the call anyway.

## Reading it

```
"Gate": "local",              // or "off"
"GateModel": "llama3.2:3b",   // same as LocalModel
"GateFollowUpSeconds": 25,
"WakeNames": "Octavia"
```

## One per room *(v0.24.0)*

Stage 14 item 7 asked whether "was that addressed to me?" needs scoping per face. The answer, once rooms existed, was per **room** — and it is a real hazard rather than a tidiness point: `_lastExchange` drives the *follow-up* rule, so a shared gate would let a conversation at the desk make a second room's gate believe it was mid-exchange and answer a stranger.

So there is one `AttentionGate` per room, constructed with the room, with no shared statics. It was **scoped rather than built out**, deliberately: only a room with always-on listening ever reaches the gate, and today that is the host room alone. Push-to-talk bypasses it entirely and always has — a held button has already answered the question the gate asks. See [[One Being, Many Rooms]].

See also [[Eyes]] — built in the same stage, and the other half of what a room-scale assistant needs.
