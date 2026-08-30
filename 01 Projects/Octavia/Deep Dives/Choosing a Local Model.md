---
project: Octavia
tags: [octavia, deepdive]
---

# Choosing a Local Model

Measured on the dev VM, 08/29/2026. The result inverted the obvious answer.

## The setup

Ollama 0.33.2, installed via winget, running as a service on `localhost:11434`. Two candidates, both through the OpenAI-compatible endpoint with her real system prompt and two ordinary questions.

Dev VM: **2 cores**, 16 GB RAM, no GPU (VMware SVGA). That is a worst case, which makes it a useful one.

## The numbers

| Model | tok/sec | Tokens per reply | **Wall clock (warm)** |
|---|---|---|---|
| `qwen3:1.7b` | **11.1** | 125–145 | 9.9 s |
| `llama3.2:3b` | 4.3 | **37–39** | **6.5 s** |

Qwen is **2.5× faster per token and still slower to finish**, because it ignores "two or three short sentences" and monologues. Llama obeys the instruction, so it stops.

Sample quality agreed: *"The weather on Mars is quite cold and dry."* against Qwen's padded equivalent.

## The lesson

**Tokens per second is the wrong metric for a talking avatar.** The user experiences wall clock, and wall clock is `tokens_emitted / rate`. A chattier model is worse twice over — slower to finish *and* she rambles. Judge on **tokens emitted**, and measure on the machine it will actually run on.

`llama3.2:3b` became the `dev` profile default.

## Why out-of-process

`LocalBrain` speaks HTTP to a separate server rather than embedding a runtime like LLamaSharp. Two reasons, and the second is the load-bearing one:

1. **Any server works.** Ollama, LM Studio, `llama-server` all expose the same shape, so swapping models or runtimes is a config edit, never a rebuild.
2. **No native-dependency collision.** An in-process runtime would put a second CUDA-linked native stack in the same process as `Whisper.net.Runtime.Cuda`, and later Audio2Face. A separate server sidesteps that entirely and can unload its model when the GPU is wanted for rendering.

On the eventual box — renderer, Whisper, Audio2Face and this all sharing one card — that flexibility stops being a convenience.

## What the host has to clean up

Small models need babysitting that Claude does not:

- **`<think>` blocks.** Reasoning models narrate their scratchpad inline; unfiltered, she reads it aloud. The tags arrive split across SSE chunks, so the filter is a state machine. It caused its own bug — see [[Lessons Learned]].
- **Markdown.** Asterisks and hashes get spoken. Flattened before the voice sees them.
- **Length.** Nothing to be done in code; it is simply the limitation.

## Setting expectations

A 4B model **is not Octavia**. It exists so that every later stage — lip sync, dancing, turn-taking, the new face — can be hammered for free and offline. Say that plainly or the dev build reads as a regression in her character.

## Its second life

This is not throwaway scaffolding. Stage 9 needs a cheap local gate in front of the expensive model to judge *"was that addressed to me, and is it worth answering?"* before spending a Claude call. That gate is this component behind this interface — which is why it earned a stage of its own rather than being a debug flag. See [[Roadmap]].
