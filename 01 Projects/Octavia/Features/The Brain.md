---
project: Octavia
tags: [octavia, feature]
---

# The Brain

*Stage 2, v0.3.0.* Claude in life; a small local model during development, and later as the gate that decides what deserves a Claude call.

## The interface

```
IBrain
├── ClaudeBrain   Anthropic SDK, streamed
└── LocalBrain    any OpenAI-compatible server, SSE
```

`IBrain` exposes `Description`, `IsReady`, `NeedsApiKey`, `Forget()` and `RespondAsync()`. `NeedsApiKey` is what keeps the key prompt from appearing when she is running on a local model — the session asks the brain rather than assuming.

## What is true right now

`RespondAsync` takes an optional **`context`** — what is happening in the room rather than what was said. Today that is only whether music is playing and roughly how fast; the camera in Stage 9 will use the same seam.

Where it goes was the whole decision. Not the **system prompt**, because the cache breakpoint sits there and anything volatile above it would void the cache on every turn. Not the **history**, because it would still be claiming there is music an hour after it stopped. So it rides with the current user message only, and is never stored — `Persona.Situated` attaches it, and both brains use the same helper so the wording never depends on which one is running.

It also tells her not to mention it unless asked. A model told "music is playing" will otherwise open every reply by saying so.

Selected by `Brain: "claude" | "local"` in config, which the `dev` and `live` profiles set — see [[Profiles & Configuration]].

## Sentence streaming

`RespondAsync` yields **one speakable sentence at a time** rather than a finished reply. The voice starts speaking on the first one while the model is still writing — and each sentence is also read for a mood on its way past, so her expression follows what she is saying. See [[The Voice]]. This is the single biggest contributor to her feeling alive, and it is why the splitter has to know that `3.50` is not two sentences.

## The persona

She is told, in short: everything she says is heard, never read. Two or three short sentences, plain words, no lists, no markdown, no stage directions. Answer first, then stop. Expect mangled words from speech recognition and answer the likely meaning rather than complaining about the transcription.

Claude follows this. A 4B local model does not — see below.

## Conversation

`Conversation` holds the running turns provider-neutrally, capped at 40 messages, and **drops whole exchanges** rather than half of one: an orphaned assistant turn at the front makes some providers reject the request outright.

A failure before the request lands drops the pending user turn, or the next request sends two user messages in a row.

## Prompt caching

`ClaudeBrain` sets a cache breakpoint on the system prompt. It is currently **inert** — the persona is a few hundred tokens and the minimum cacheable prefix is over a thousand. It starts paying once the persona, memory and tool definitions grow, with no code change. Anything volatile must stay *after* the breakpoint; a timestamp in the system prompt would silently kill it.

## Local model quirks the host absorbs

Small models ignore instructions that Claude follows, so `LocalBrain` cleans up after them:

- **`<think>` blocks are stripped** before the voice sees them, by a streaming state machine — the tags arrive split across SSE chunks. See [[Lessons Learned]] for the bug this caused.
- **Markdown is flattened.** Otherwise she reads asterisks and hashes aloud.
- **Failures are actionable**: *"No local model server at http://localhost:11434/v1. Start Ollama or LM Studio."* rather than a raw connection exception.

## Why the local model stays after development

Stage 9 needs a cheap gate in front of the expensive model — something to judge *"was that addressed to me, and is it worth answering?"* before spending a Claude call. That is this component and this interface. Building it now means always-on listening later is a config change, not a new subsystem. See [[Roadmap]].

## Honest limitation

**A 4B model will not hold her persona.** Expect rambling and formatting no matter what the system prompt says. Judge lip sync, latency and turn-taking on the local brain; judge *Octavia* on Claude. See [[Choosing a Local Model]].
