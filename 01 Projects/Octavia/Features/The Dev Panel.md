---
project: Octavia
tags: [octavia, feature]
---

# The Dev Panel

*v0.8.1.* Every performance the face can give, on a button.

## The problem it solves

The face does what the host tells it, which is correct and makes anything rare hard to look at: to see the `surprised` expression you have to make her surprised, and to see the headphones you have to play music she agrees is music. By the time the state you wanted arrives it has usually moved on.

So the panel drives `window.Face` **directly**. Nothing goes through the host, nothing is faked at the host end, and a shape can be held still long enough to argue about.

This is the same instinct as the probes in `tools\EarsTest` — see [[Diagnostics]]. Some things can only be judged by looking, and the tooling has to let you look.

## What is on it

| Group | Drives |
|---|---|
| The host | `Hold the face` — see below |
| State | idle / listening / thinking / speaking |
| Mood | the six VRM expressions, with a weight slider |
| Mouth | the five visemes and `shut`, an openness slider, and `Say a line` |
| Eyes | a blink on demand, five gaze directions, and `Let her look` to give them back |
| Microphone level | the amplitude the halo and the plinth arc answer |
| Music | playing, tempo, energy, a single beat, and beats at the tempo |
| Props | headphones `auto` / on / off |
| Room | the day's keyframe hours |
| Senses | listening, hush, and the music sense — **these reach the host** |

`Say a line` exists because a single held viseme says nothing about whether a mouth reads as *talking*. Sequence is most of the effect, and this is the cheapest way to see it without a voice.

## Hold the face

The one idea worth explaining. While it is on, host messages that would **move** her — `state`, `level`, `viseme`, `emotion`, `music` — are dropped in `bridge.js` before they reach the renderer. Everything that is *information* — captions, the transcript, notices, `hello` — still arrives.

Without it the panel is nearly useless: a mood set by hand is wiped by the next thing she says, and the headphones come straight back off the moment the host sends a `music` update. With it, the panel is authoritative and the host is a spectator.

## Senses are different, deliberately

Every other control stays in the renderer. The Senses row does not, and it is labelled so, because a microphone and a loopback are **devices**, and the face does not own one — that is the whole architecture, not an implementation detail. See [[Architecture]].

The alternative would have been host-side debug commands for everything, which would have meant a debugging surface in the being, reachable by any face. This way the only thing the panel can ask the host for is something a normal face can already ask for.

## When it appears

- On the **`dev` profile**, via a `dev` flag in `hello`.
- Whenever the face is **served without a host** — a face on its own is being worked on by definition, so `tools\serve-face.ps1` gets it for free. See [[Build & Release]].
- `DevPanel` in `config.json` forces it either way.

The module is a dynamic `import()`, so a published face on the `live` profile never fetches it. The button is `hidden` until the host says otherwise.

## What it added to the face's surface

Three handles, for things the face schedules *for itself* and so could not otherwise be asked for:

```js
Face.blink()                              // now, then back to the ordinary schedule
Face.look(x, y)                           // hold a gaze; Face.look(null) releases it
Face.setProp('headphones', on)            // null hands the prop back to the music
```

Plus `Face.read()`, so opening the panel shows what is currently true rather than resetting her to whatever the controls happened to default to.

`setProp` is the interesting one: the headphones already had an owner — the music — and a second owner needed a way to say "not me" as well as on and off. `null` for *auto* rather than a separate boolean keeps that a single control with three positions, which is what it actually is.
