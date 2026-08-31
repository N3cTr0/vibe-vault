---
project: Octavia
tags: [octavia, feature]
---

# The Ears

*Stage 1, v0.2.0.* Microphone → Silero VAD → Whisper, entirely local. Nothing acoustic leaves the machine.

## The chain

`WhisperRecognizer` implements [[Architecture|ISpeechRecognizer]], so it dropped in beside the Windows recognizer without anything else changing.

1. **Capture** — NAudio `WaveIn` at 16 kHz mono, 32 ms buffers (the rate Silero and Whisper both want, so no resampling).
2. **Gate** — Silero VAD scores every 512-sample frame. See [[Silero VAD Context Window]] for the trap.
3. **Segment** — hysteresis around the score:
   - start at p ≥ 0.50, end at p < 0.35 (separate thresholds stop flapping mid-word)
   - **320 ms of pre-roll** kept from before speech began, so word onsets are not clipped
   - **800 ms of quiet** ends the utterance
   - under 300 ms of voiced audio is discarded as a noise blip
   - 30 s hard cap
4. **Transcribe** — only audio the VAD vouched for reaches Whisper. See [[Whisper Integration]].
5. **Filter** — segments Whisper itself flags as probably-not-speech are dropped, as are bracketed tags like `[BLANK_AUDIO]`.

## Why the VAD is not optional

Whisper hallucinates fluent text out of silence — it will confidently produce "Thank you." or subtitle credits from room noise. Bolting Whisper on without a gate makes her *worse* than the Windows recognizer, because she starts answering the hum of the PC. The VAD is the thing that makes Whisper usable, not a performance optimisation.

## Not hearing herself

`OctaviaSession` mutes the recognizer for the duration of her reply and unmutes only when the brain stream has ended *and* the speech queue has drained. Muting also resets the VAD state, so a half-captured utterance is not resumed afterwards.

## The silence watchdog

A capture device can open successfully and deliver pure digital silence — a muted input, or RDP with microphone redirection off. That failed *invisibly*: she looked like she was listening and simply never answered.

Now: mic open, peak amplitude at exactly zero for 10 seconds → a notice on her face naming the RDP setting. The floor is 0.0005, which distinguishes digital silence from a quiet room (a live mic always has a noise floor), so it does not cry wolf.

## Where the audio comes from *(v0.23.0)*

Until Stage 14 item 2, `WhisperRecognizer` did not *consume* microphone audio — it **was** the microphone. It built its own `WaveIn`, chose the device, and raised into its own frame loop, so there was nothing to hand a different source to. `IAudioSource` is that seam: `LocalMicSource` is today's capture lifted out unchanged, `FaceAudioSource` is fed by binary frames from a phone. Everything from the 512-sample frames inward never cared where they came from and is untouched.

**Push-to-talk, and it removes work rather than adding it.** A held button has already answered *"was that addressed to me?"*, so [[The Attention Gate]] does not apply to that path. One talker at a time also means one Whisper — the earlier worry about two concurrent transcriptions on eight cores does not arise. And a released button is an *exact* end-of-utterance marker, so `Flush()` transcribes immediately instead of waiting 800 ms for the detector to guess.

### Two things it would be easy to get wrong

Both were flagged in the spec, both were real, and both would have looked like something else entirely.

- **The room-music analyser is fed from the microphone.** It used to hang off `whisper.Audio`, which was right while there was one microphone in the world. Swap the source and that subscription quietly follows the phone — she would report the tablet's kitchen radio as the music around *her*, with everything still appearing to work. So the local microphone is owned by the session, shared, and framed **separately** for music. It keeps running while a face holds the floor: speech moves rooms, her sense of what is playing here does not. See [[Music]].
- **The deaf-microphone warning would cry wolf.** For a local device, silence means broken and saying so is valuable. For push-to-talk, silence is the normal state — nobody is holding the button — so every remote session would have named Remote Desktop audio settings at somebody holding a phone. It is gated on `ExpectsContinuousAudio`, which is why that flag is on the interface rather than inferred.

The rule that `UseSource` detaches the old source but does **not stop** it is a *test*, not a comment, because it is exactly what a later tidy-up would undo.

## Configuration

| Key | Default | Notes |
|---|---|---|
| Key | Default | Notes |
|---|---|---|
| `Recognizer` | `whisper` | `whisper` or `windows` |
| `WhisperModel` | `large-v3-turbo` | `small.en` in the `dev` profile |
| `WhisperLanguage` | `en` | ISO code, or `auto` to detect per utterance |
| `WhisperCompute` | `auto` | `auto` / `cpu` / `gpu` — see below |
| `WhisperThreads` | `0` | 0 lets Whisper choose |
| `MicrophoneDevice` | *(empty)* | Substring of the device name; empty is the Windows default |
| `MinConfidence` | `0.35` | Raise if she answers the television |

The Windows desktop recognizer remains an automatic fallback: if Whisper fails to start, she logs it, says so, and carries on with the old ears.

**`WhisperCompute` exists because "auto" is not neutral.** Whisper loads the GPU runtime wherever one will load at all, and on a weak card that is *slower* than a good processor — which is exactly the machine she moved to. The setting is exposed in Settings as "Speech recognition runs on", and it takes effect on restart because the runtime order is chosen when the library loads.

**`MicrophoneDevice` matches on a substring**, not an exact name, because `WaveIn` truncates device names to 31 characters while the WASAPI enumeration does not. Matching the full string would silently fail for any device with a long name.

## Testing

`dotnet run --project tools\EarsTest` synthesizes speech with SAPI, runs the whole chain, and asserts both that the phrase comes back and that three seconds of silence produces nothing. `-- mic` enumerates capture devices and reads Windows' own meter — see [[Lessons Learned]].

## Not yet

No wake word. No speaker identification.

*The line that used to be here — "continuous listening sends every recognised phrase to the brain, so always-on is a demo rather than a mode to live in" — has been **untrue since v0.9.0**.* A small local model now judges what was addressed to her before the brain is troubled, and what it declines is shown faintly rather than swallowed. See [[The Attention Gate]].
