---
project: Octavia
tags: [octavia, deepdive]
---

# The Photoreal Decision

*Stage 8, decided 08/30/2026.* Which way she becomes photoreal, and why — written down while the reasoning is fresh, because this is the decision most likely to be re-argued later.

Stage 8 was always a **decision gate**: the roadmap deliberately refused to choose in advance, on the grounds that the options would move. They did.

## The three candidates

| Route | What it is | Verdict |
|---|---|---|
| **MetaHuman in Unreal** | A rigged, scanned character rendered by UE, attached over the Stage 3 socket | **Chosen** for rendering |
| **Audio2Face-3D** | NVIDIA's audio → ARKit-blendshape inference, MIT licensed | **Chosen** for animation |
| **Gaussian splatting** | Millions of 3D splats reconstructing a real head | Rejected, for now |

## Why MetaHuman, when it is not the most photoreal

This is the part worth remembering, because on the narrow question of *looking like a person* MetaHuman loses. Splatting-based heads are better, and honest write-ups say MetaHuman still reads as "a very good game character".

It wins on a requirement that only exists because of [[Music]]: **she wears headphones.**

A rigged character can hold a prop, turn its head, take a pose, and be told to look left. A neural head reconstructed from video can do none of those things — there is no bone to attach to and no vocabulary to direct it with. The roadmap predicted this in Stage 5 ("a photoreal neural face could never hold a prop"), and Stage 7 turned it from a prediction into a working feature that a new renderer would have to *lose*.

An assistant who cannot put her headphones on is a worse Octavia than a slightly less real one. That is the whole argument.

**Gaussian splatting is not rejected forever.** It is genuinely the better-looking option and the research is moving fast. It is rejected because it is research-grade and undirectable. Revisit when it can be told to look left.

## Why Audio2Face, and the licensing that made it possible

- **MIT licensed**, with a C++/CUDA SDK that does **local inference** — no cloud, nothing acoustic leaving the machine, which is a standing constraint rather than a preference. See [[Conventions & Security Model]].
- Emits **ARKit blendshapes**, which is what a MetaHuman speaks natively. No translation layer, the same reasoning that made the protocol's expression vocabulary VRM 1.0's in Stage 5.
- Better than 60 fps generation. Requires **CUDA 12.8+, TensorRT 10.13+, 4 GB+ of card**.
- Ships a training framework, so a rig can be fine-tuned later rather than taken as given.

## The decision inside the decision: where Audio2Face runs

NVIDIA ships an **Unreal plugin**, which makes putting it in the renderer look obvious. It is the wrong choice, and the reason is a piece of Stage 6.

**The host owns the voice.** It synthesises the audio and plays it, and Stage 6 already put a tap — `MouthTap` — between the buffer and the sound card, because lip sync had to be measured at the moment sound is *heard* rather than generated. The PCM is already in hand, at exactly the right instant, in the host.

Run Audio2Face there and:

- audio never has to cross the protocol, so the face still owns no audio device
- **every** face benefits, including the VRM one that exists today
- `OctaviaSession` keeps its shape — the blendshapes go out as another message

Run it in the renderer and the host has to ship PCM over a WebSocket to something that then plays or analyses it, which breaks the cleanest line in the architecture. See [[Architecture]].

**Out of process, though.** In-process it would be a second CUDA runtime beside Whisper's, which the standing constraints forbid. That makes it the Piper arrangement exactly — a long-lived child fed on stdin — and Piper has been running since Stage 6 without trouble. See [[The Voice]].

## The dated constraint nobody knew about

**The MetaHuman Creator web app shuts down on 11/05/2026.** Creation moves into Unreal Engine 5.7's in-editor tooling. That is about nine weeks from this decision.

It does not change the choice — the plan was always an Unreal application — but it does mean a character made the browser way has to be made and exported before then. Worth knowing now rather than in November.

## What was actually built

The stage's premise was that photorealism becomes *attaching a different renderer*. That was an assumption. It is now tested.

`tools\attach-face.ps1 -Conformance` attaches as an external face, drives a real turn plus a self-test and a forget, and reports which host-to-face messages arrived and whether each carried the fields the protocol promises. `PROTOCOL.md` gained **What a renderer must implement** — must-handle, may-ignore-and-what-it-costs, and the message rates a renderer has to survive.

**It found a real gap on its first run**, which is the pattern every harness in this project has followed:

> A face attaching to a session already in progress was never told what expression she was wearing.

`emotion` is only sent when her mood *changes* — correct, and the reason it is correct is in [[The Face]] — but that means a renderer connecting mid-conversation had nothing to go on, and a mood can sit unchanged for many minutes. An Unreal face would have shown the wrong expression indefinitely and it would have looked like a bug in the renderer.

`hello` now carries `state`, `emotion` and `emotionWeight`. The fix is four lines. Finding it without a renderer to find it with is the point.

## What is blocked, and honestly so

> **The move to a physical PC may unblock this entirely.** Nothing about the decision changes — only whether it can be acted on. Check the new machine against the shopping list at the bottom of this note, and mind the **11/05/2026** MetaHuman Creator web-app shutdown. See [[Moving To The New Machine]].

The rendering. This VM reports a **VMware SVGA 3D** adapter, three cores, no CUDA. Unreal Engine 5.7, a MetaHuman and Audio2Face all need a real NVIDIA card, and none of them can be even installed here, let alone judged.

So the stage delivers the decision, the contract and the fix, and stops. Building an Audio2Face integration with no GPU to run it against would produce code nobody could verify — which is precisely the kind of work [[Lessons Learned]] says not to do.

## The shopping list, when the machine exists

- Unreal Engine **5.7** or later (MetaHuman creation is in-editor from here on)
- An NVIDIA card with **4 GB+** free after Whisper and the local gate have taken theirs
- **CUDA 12.8–12.9**, **TensorRT 10.13+**
- Budget the card as: renderer + Whisper + Audio2Face + local gate, sharing one GPU
