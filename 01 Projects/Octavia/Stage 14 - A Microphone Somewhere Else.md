---
project: Octavia
tags: [octavia, spec, stage-14]
---

# Stage 14, item 2 — A microphone somewhere else

> **A specification, not a change.** Written 08/31/2026 from the [[Octavia Android]] side, for whoever is working in her repo. Nothing in her repo was touched to produce it. Third of the series, after [[Stage 14 - Face Identity]] (v0.21.0) and [[Stage 14 - Her Voice On Another Face]] (v0.22.0). This is the last thing between a tablet and a peer.

## Why this is the big one

Items 1 and 3 were plumbing: a seam repaired, and a stream teed from a point that already existed. **This one is a refactor of how she hears.**

`WhisperRecognizer` does not *consume* microphone audio — it **is** the microphone. It constructs its own `WaveIn`, sets the format, picks the device, and raises `DataAvailable` into its own frame loop. `MicLevelMeter` does the same, separately. There is no seam to hand a different source to, so one has to be made.

The good news is that the shape is already right one layer in: `OnAudio` converts bytes to floats and hands **512-sample frames at 16 kHz** to the VAD. Everything above that line is source-agnostic already.

## The seam

```csharp
namespace Octavia.Senses;

/// Where 16 kHz, 16-bit, mono, little-endian PCM comes from. The ears stop caring which
/// room it was recorded in.
internal interface IAudioSource : IDisposable
{
    /// A buffer and how much of it is real. Same contract as `WaveInEventArgs`, so the
    /// existing frame loop needs no changes at all.
    event Action<byte[], int>? Data;

    /// What this is, for the log and for `hello`: a device name, or a face.
    string Name { get; }

    /// Whether silence from this source is worth complaining about. See *Two traps*.
    bool ExpectsContinuousAudio { get; }

    void Start();
    void Stop();
}
```

- **`LocalMicSource`** — today's `WaveIn` and `AudioDevices.WaveInIndex` lifted out of `WhisperRecognizer` unchanged. `ExpectsContinuousAudio => true`.
- **`FaceAudioSource`** — fed by binary frames from one face. `ExpectsContinuousAudio => false`.

`WhisperRecognizer` takes an `IAudioSource` and keeps everything from `OnAudio` inward. `MicLevelMeter` should take one too, or the level meter will still be opening the local device while the ears listen to a phone.

**The format is fixed by contract, not negotiated.** 16 kHz because that is what Silero and Whisper want, and the phone has cycles to spare for the resample — the host should not grow a resampler to save a handset some work.

## Push-to-talk, and what it buys

The client holds a button; audio flows only while it is held. This was decided from the hardware ([[Octavia Android]]), but it earns three things that are worth being explicit about, because each one removes work:

1. **The attention gate does not apply to this stream.** `AttentionGate` exists to decide *"was that addressed to me?"* — and a held button has already answered. Item 7 of this stage (the gate across two rooms) is **not needed for push-to-talk** and should not be pulled in.
2. **One talker at a time means one Whisper.** An earlier note of mine warned about sizing two concurrent transcriptions against 8 cores. With a single floor-holder that concern disappears — **disregard it**; there is never a second stream.
3. **No echo cancellation.** She is not speaking while the button is held — and if she is, holding it is an interrupt, not an echo. See *Barge-in* below.

## Protocol

### Audio upstream

**Binary frames, mirroring the downstream rule already in `PROTOCOL.md`:** a binary frame is audio and nothing else, in both directions. No header. 16 kHz, 16-bit, mono, little-endian, fixed.

### Holding the floor

A new face→host message, because the host must know a stream is starting and, more importantly, when it has stopped:

```json
{ "type": "talking", "value": true }   // holding the button
{ "type": "talking", "value": false }  // released — transcribe what you have
```

`listen` is **not** the right message to reuse: it toggles *her own* microphone, which is a different thing that must keep working independently.

`value: false` is the end-of-utterance marker, which is a real simplification — the VAD no longer has to guess where the sentence stopped for this source. A dropped connection means the same thing: **treat a face that vanishes mid-stream as a release**, or the floor is held forever.

### Telling a face it can

`hello` gains `micAccepted` — whether the host will take audio from a face at all. A face should not offer a microphone button that could only fail, exactly as `camera` already lets it hide controls.

## Arbitration

One talker at a time, and the host decides:

- While a face holds the floor, **the local microphone is muted** — otherwise she hears the desk room and the tablet room at once and transcribes a blend of both. `Mute()`/`Unmute()` already exist for the barge-in case and are the right lever.
- A second face pressing while another holds the floor is **refused**, with a `notice` back to it rather than silence. First press wins; the alternative is two half-sentences interleaved into one transcript.
- The floor is released by `talking:false`, by disconnection, or by a timeout — a phone in a pocket with a stuck button should not own her ears indefinitely.

## Barge-in

Holding the button while she is speaking should **hush her**, not be recorded over the top of her. That is what a person means by talking over her, and `hush` already does exactly the right thing. Send it on press when `state == speaking`.

## Two traps

Both are live, and both would produce behaviour that looks like something else entirely.

### 1. The room-music analyser is fed from the microphone

`OctaviaSession.cs:777` — `whisper.Audio += _roomMusic.Push`. Those mic frames feed Stage 11a, so she can hear music playing **in this room** rather than only what this machine plays.

If the source is swapped wholesale, **the room analyser starts analysing the phone's room.** She would report the tablet's kitchen radio as the music around her. The fan-out has to stay bound to the *local* source, not to whatever the recogniser currently consumes — which probably means the local source keeps running for music even while a face holds the floor for speech, and the two are separated rather than switched.

This is the single most likely thing to be got wrong, because everything will appear to work.

### 2. `WatchForSilence` will cry wolf

The deaf-microphone warning — *"The microphone is open but completely silent"* — is correct and valuable for a local device that is open and delivering nothing. For a push-to-talk face, **silence is the normal state**: nobody is holding the button.

Gate it on `ExpectsContinuousAudio`, or every remote session raises a `Trouble` that names RDP settings at someone holding a phone.

## Explicitly not in scope

- **No always-on listening from a face.** That needs real echo cancellation and is its own piece of work, noted in [[Octavia Android]] as needing `arm64-v8a` *and* `armeabi-v7a` for the J7.
- **No attention gate changes** (item 7). Push-to-talk does not need it.
- **No turn ownership** (item 5). The floor-holder is a simpler idea and does not have to grow into one.
- **No codec.** Raw PCM upstream too — 16 kHz mono is 32 KB/s, less than her voice going the other way.

## Acceptance

1. **A held button on a face produces a transcript**, and she answers it.
2. `talking:false` ends the utterance without waiting for the VAD to decide.
3. **A face that disconnects mid-stream releases the floor**, and the local microphone comes back.
4. A second face pressing while another holds the floor gets a `notice`, and the first stream is unaffected.
5. **The local microphone is muted while a face holds the floor** — say something at the desk during a remote utterance and it must not appear in the transcript.
6. **The room-music analyser still hears this room** while a face is talking. The trap above, made testable.
7. No `Trouble` about a silent microphone during a normal remote session.
8. `hello` reports `micAccepted`.
9. Pressing while she is speaking hushes her.

`tools\EarsTest` can cover 2, 3, 4 and 7 with a synthetic face pushing a canned WAV, which is also the fastest way to iterate without a handset.

## What the phone will do

- `RECORD_AUDIO` at runtime, and a mic button that is only shown when `micAccepted`.
- `AudioRecord` at 16 kHz mono 16-bit — **native, not a WebView**, since `getUserMedia` will not run on a plain `http://` LAN origin.
- Resample only if the device refuses 16 kHz; most will not.
- Send `talking:true`, stream binary frames while held, `talking:false` on release — and on losing the link, so the floor is never left held.
- Keep skipping `viseme` and `level`; a microphone is not a reason to start receiving those.

## Links

- [[Stage 14 - Face Identity]] · [[Stage 14 - Her Voice On Another Face]]
- [[Roadmap]] — *Stage 14*, for the remaining items
- [[The Ears]] — how she hears today
- [[The Attention Gate]] — and why push-to-talk does not need it
- [[Octavia Android]] — the client that needs this
