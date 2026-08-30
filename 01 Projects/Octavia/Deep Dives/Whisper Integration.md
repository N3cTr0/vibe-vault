---
project: Octavia
tags: [octavia, deepdive]
---

# Whisper Integration

`Whisper.net` 1.9.1 over whisper.cpp. CPU and CUDA runtime packages are both referenced; CUDA is selected automatically when present.

## Why local, not cloud

The prototype's Web Speech API was not doing recognition in the browser — it streamed the microphone to Google's servers. For something that sits in a room listening, local is the right answer on both quality and principle. Whisper is MIT, weights included, and handles accents, background noise and half-sentences far better than the Windows desktop recognizer. It also handles Afrikaans, which the Windows recognizer almost certainly has no engine for.

## Model choice

| Model | Size | Use |
|---|---|---|
| `small.en` | ~466 MB | The `dev` profile — quick on a CPU-only box |
| `large-v3-turbo` | ~1.6 GB | **The default.** ~1–2% worse WER than large-v3, several times faster |
| `large-v3` | ~3 GB | A config flip away when accuracy beats latency |

Turbo is the default not because the GPU cannot run large-v3 — it easily can — but because that same GPU will later render her face and run Audio2Face. Turbo leaves headroom. Roughly 10 GB VRAM for large-v3 against 6 GB for turbo in common implementations.

Models download once to `<data>\models`, written to a `.partial` file and renamed on completion so an interrupted download never leaves a corrupt model behind. Progress is reported to her face; the first listen can take a while and must never block the message loop, so the whole open-ears path is async.

## Two API traps

**Confidence is zero unless you ask for it.** `SegmentData.Probability` reads `0.00` on every segment unless the processor is built with `.WithProbabilities()`. A confidence gate silently rejecting everything looks exactly like a broken recognizer.

**Native teardown corrupts the process exit code.** The test harness returned normally with every check passing and the process reported exit −1. `Environment.Exit(n)` rather than `return n`.

## Hallucination defences

Whisper will produce fluent text from silence. Three layers, in order of importance:

1. **The VAD gate.** Whisper only ever sees audio Silero vouched for. This is the one that matters — see [[Silero VAD Context Window]].
2. **Whisper's own tell.** Segments with `NoSpeechProbability > 0.6` and `Probability < 0.4` are dropped.
3. **Bracketed tags.** Anything wrapped in `[...]` or `(...)` — `[BLANK_AUDIO]`, `(wind blowing)` — is non-speech annotation, never something to say back.

The regression test asserts that three seconds of digital silence transcribes to the **empty string**. Before layer 3 it returned `[BLANK_AUDIO]`, which would have been captioned and answered.

## Threading

Transcription runs off the audio callback — the capture thread must never block, or buffers drop. A `SemaphoreSlim(1,1)` serialises transcription so two overlapping utterances cannot enter whisper.cpp concurrently. Results are discarded if she was muted or stopped listening while the transcription was in flight.

## Testing

`tools\EarsTest` synthesizes a known phrase with SAPI at exactly 16 kHz mono, pushes it through the real classes, and asserts both the transcript and the silence case. Even `tiny.en` returns the phrase perfectly, which makes the harness fast enough to run on every change.
