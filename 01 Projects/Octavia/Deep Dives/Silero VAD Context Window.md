---
project: Octavia
tags: [octavia, deepdive]
---

# Silero VAD Context Window

The single most expensive hour of Stage 1, and a trap anyone porting Silero outside Python will hit.

## The symptom

Clean 16 kHz speech in, `0.00` out. For every frame. No exception, no warning, no NaN — the model ran happily and returned confident silence on obvious speech. Silence also scored `0.00`, so the numbers looked *self-consistent* rather than broken.

## The diagnosis

Rather than reading more documentation, the probe fed the same audio twice — once as bare 512-sample frames, once with 64 samples of the preceding audio prepended:

```
context=0:  0.00 0.00 0.00 0.00 0.00 0.00 ...
context=64: 0.36 0.91 1.00 1.00 1.00 1.00 ...
```

Unambiguous in one run.

## The cause

The ONNX graph's `input` tensor is declared `[-1, -1]` — fully dynamic — so passing exactly 512 samples is *structurally valid* and fails silently. The model actually expects **64 samples of context plus the 512-sample frame**: 576 values, with the context being the tail of the previous frame.

The official Python wrapper maintains that context buffer internally, so it never appears in the model signature or in most examples. Read only the ONNX metadata and you get this wrong every time.

## The fix

`SileroVad` keeps a 576-float input buffer. Each call writes the new frame at offset 64, runs inference, then copies the frame's **last 64 samples** to the front for next time. `Reset()` clears both that buffer and the recurrent `state` tensor.

```csharp
private const int ContextSamples = 64;
Array.Copy(frame, 0, _input, ContextSamples, FrameSamples);
// ... run ...
Array.Copy(frame, FrameSamples - ContextSamples, _input, 0, ContextSamples);
```

## Also worth knowing

- The `state` tensor `[2, 1, 128]` is recurrent — carry `stateN` forward between frames, and clear it whenever listening stops or the recognizer is muted, or a stale utterance bleeds into the next one.
- The `sr` input is a **scalar** (`Int64 []`, zero dimensions) in the current model, not a 1-element vector. `SileroVad` checks the metadata and shapes it accordingly rather than assuming.
- 512 samples at 16 kHz is 32 ms, which is why the capture buffer is set to 32 ms — one buffer, one frame, no reassembly.

## The general lesson

**A dynamic tensor shape means the model cannot tell you that you are wrong.** When an ML component returns plausible-but-constant output, suspect the input contract before the logic, and prove it by varying one thing at a time against a known-good signal. See [[Lessons Learned]].
