---
project: Octavia
tags: [octavia, deepdive]
---

# The Avatar Interface

*Stage 5, v0.6.0.* The seam that lets a plaster bust and a photoreal character take the same performance.

## The same bet, one level down

Stage 3 made the *protocol* the interface, so anything that speaks it is a legal face — see [[The Host-Face Bridge]]. That worked, so Stage 5 made the same bet inside the renderer: **the face is a performance; the avatar is a puppet.**

```js
{
  root,                       // an Object3D the scene adopts
  setViseme(shape, weight),   // 'aa'|'ih'|'ou'|'ee'|'oh'|null
  setExpression(name, weight),
  setGaze(x, y),
  setBlink(v),
  setPose(yaw, pitch, roll),
  setLevel(v),
  setHeadphones(v),           // 0 off and away, 1 on her head
  setBreathing(on),
  update(dt, elapsed)
}
```

`setHeadphones` arrived in Stage 7 and is the first thing on this list that is an *object* rather than a part of her. It went here rather than into `face.js` for the same reason as the rest: where the prop hangs is a fact about the character — a unit-sphere bust and a VRM head fifteen centimetres across need the same headphones at very different sizes, and only the avatar knows which it is. The geometry itself is one shared module, built to a radius it is told. See [[Music]].

`face.js` owns everything about *how she performs*: when she blinks, where her eyes dart while thinking, how her head carries in each state, how a mood eases in. `bust.js` and `vrm-avatar.js` own only *how a jaw actually moves*. Swapping between them is one ordinary function:

```js
function adopt(next, nextFrame) { … }
```

Which is the point. Stage 8's photoreal renderer arrives the same way.

## Why the vocabulary is VRM's

The expression names are not ours. They are the **VRM 1.0 presets** — `happy`, `angry`, `sad`, `relaxed`, `surprised`, `neutral` — and the visemes are VRM's five: `aa`, `ih`, `ou`, `ee`, `oh`.

Choosing them meant the chain from a sentence to a character's blendshape is an **identity mapping**, with no translation table anywhere. Verified against a real VRM: every name the protocol sends exists on the model, and setting them reads back exactly.

A vocabulary invented first would have needed a lookup, and a lookup is where this sort of thing rots — one name gets added on one side, and a face quietly stops smiling.

## Visemes gained a shape

SAPI reports 21 viseme identifiers. Until Stage 5 the host collapsed them to one number: how far the jaw drops. That number is genuinely most of lip sync, but it makes "aa" and "ou" identical — same drop, and no lips.

So `Shape(int)` joins `Openness(int)` in [[The Voice]], mapping the same identifiers onto the five VRM mouths. The protocol's `viseme` message carries both, and `shape` is optional so a face written before it keeps working.

Even the bust uses it: its mouth quad is scaled per shape (wide for `ee`, round for `ou`), which is more expressive than one openness value on a renderer that is about to be retired anyway.

## Where mood comes from

`Moods.Read(sentence)` reads an expression out of the text as she speaks it — locally, free, no model call. It obeys the standing rule that reflex-speed things stay local and the model is for thought.

It is deliberately conservative: a clear cue ("I'm sorry", "glad to", "wow") or nothing. Punctuation colours her only slightly, because a face that beams at every exclamation mark reads as unhinged. The `emotion` protocol message exists so a better brain can override it later.

The mood is sent **only when it changes** — an expression is a movement towards a face, and repeating it every sentence would keep restarting that movement. It resets to neutral when the turn ends, so the last sentence's mood does not linger.

## Loading a character

A `.vrm` goes in `<data>\avatars`, named in `AvatarFile`. The host maps that one folder to a read-only `https://octavia.avatar` origin and puts the URL in `hello` — the face never reads an arbitrary path, and the CSP names that origin explicitly.

The face loads it **once** (a VRM is megabytes; every later `hello` would refetch it) and falls back to the bust on any failure, reporting through `faceError` so it reaches the log. "She looks wrong" is otherwise the entire bug report; the self-test also has an **Avatar** check that turns it into a filename. See [[Diagnostics]].

Two things every VRM needs on arrival:

- **A pose.** The format defines a rest position — a T-pose — not an idle. Arms come down on load.
- **A camera.** Rather than scaling a person to fit the bust's framing, the avatar reports where its own head bone is and the face frames *that*. A tall character and a short one end up the same size on screen.

## Choosing one (v0.6.1)

**Settings → Appearance** lists the plaster bust plus whatever `.vrm` files are actually in the folder, so picking a character is a dropdown rather than a filename typed into a config file. The host refuses a name that is not there rather than saving it, and the choice applies immediately.

Three things that only showed up once a menu existed:

- The face **only ever loaded** an avatar; it could not switch back to the bust, because the first version compared against "have we loaded anything yet" rather than "is this the one showing".
- A face that cannot reach the avatar origin at all — any face outside the app — retried on **every** `hello`, which a settings menu sends constantly. A URL that failed is now not retried for the rest of the session.
- The settings did not persist at all. That one belongs to [[Profiles & Configuration]], and it is the better story.

## What is still missing

Headphones as an attached prop — the reason a stylised rig was chosen over a neural face in the first place — wait for [[Roadmap]] Stage 7, when there is music to put them on for. And Octavia has no character file of her own yet: that is a design decision, not an engineering one.
