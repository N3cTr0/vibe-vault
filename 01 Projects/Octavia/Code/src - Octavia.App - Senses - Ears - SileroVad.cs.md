---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Ears\SileroVad.cs
---

# src\Octavia.App\Senses\Ears\SileroVad.cs

```csharp
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;

namespace Octavia.Senses;

/// Silero VAD: 32ms of 16kHz audio in, speech probability out. This tiny model is the
/// gate that keeps Whisper from transcribing silence into hallucinated text.
internal sealed class SileroVad : IDisposable
{
    public const int SampleRate = 16000;
    public const int FrameSamples = 512;

    // The model wants each frame prefixed with the tail of the previous one; without
    // this context it returns zero for everything, silently.
    private const int ContextSamples = 64;

    private readonly InferenceSession _session;
    private float[] _state = new float[2 * 1 * 128];
    private readonly float[] _input = new float[ContextSamples + FrameSamples];
    private readonly bool _scalarSr;

    public SileroVad(string modelPath)
    {
        _session = new InferenceSession(modelPath);
        _scalarSr = _session.InputMetadata.TryGetValue("sr", out var sr) && sr.Dimensions.Length == 0;
    }

    /// frame must be exactly FrameSamples mono floats in [-1, 1].
    public float Probability(float[] frame)
    {
        if (frame.Length != FrameSamples)
            throw new ArgumentException($"expected {FrameSamples} samples, got {frame.Length}");

        Array.Copy(frame, 0, _input, ContextSamples, FrameSamples);

        var inputs = new List<NamedOnnxValue>
        {
            NamedOnnxValue.CreateFromTensor("input", new DenseTensor<float>(
                (float[])_input.Clone(), [1, _input.Length])),
            NamedOnnxValue.CreateFromTensor("state", new DenseTensor<float>(_state, [2, 1, 128])),
            NamedOnnxValue.CreateFromTensor("sr", new DenseTensor<long>(
                new long[] { SampleRate }, _scalarSr ? [] : [1]))
        };

        using var results = _session.Run(inputs);
        foreach (var r in results)
            if (r.Name == "stateN")
                _state = r.AsEnumerable<float>().ToArray();

        Array.Copy(frame, FrameSamples - ContextSamples, _input, 0, ContextSamples);

        return results.First(r => r.Name == "output").AsEnumerable<float>().First();
    }

    public void Reset()
    {
        Array.Clear(_state);
        Array.Clear(_input);
    }

    public void Dispose() => _session.Dispose();
}
```
