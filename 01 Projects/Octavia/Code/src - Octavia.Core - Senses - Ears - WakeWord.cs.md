---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Ears\WakeWord.cs
---

# src\Octavia.Core\Senses\Ears\WakeWord.cs

```csharp
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;
using Octavia.Core;

namespace Octavia.Senses;

/// *"Hey Octavia"* — the thing that decides whether anything gets transcribed at all.
///
/// **She already decided correctly and paid far too much to do it.** `AttentionGate` answers
/// *was that meant for me*, and it answers it well — but only after Whisper has turned the
/// utterance into words, and since Stage 14 item 6 that is every utterance in a listening
/// room, indefinitely, on eight CPU threads. This moves the always-on layer from a 1.6 GB
/// speech model to about 3.7 MB of ONNX, and nothing is transcribed until she is addressed.
///
/// **It runs on the server, which is not a contradiction of the device rule.** That rule is
/// about *devices*; this is arithmetic on a stream faces are already sending, the same as
/// Whisper and the gate. Nothing new is opened.
///
/// ### Three models, in a chain
///
/// openWakeWord is not one network. Audio goes through a melspectrogram front end, then a
/// shared speech-embedding backbone, and only the last and smallest model knows which phrase
/// it is listening for. The first two are common to every wake word there is, which is why a
/// second phrase later costs a megabyte rather than four.
///
///   16 kHz PCM ──▶ melspectrogram ──▶ 76×32 window ──▶ embedding ──▶ 96 floats
///                                     (slide by 8)                      │
///                                                                       ▼
///                                        last 16 embeddings ──▶ classifier ──▶ score
///
/// The magic numbers are the model's, not choices: 76 mel frames in, slide 8, keep 16
/// embeddings of 96. The `/10 + 2` on the melspectrogram output is likewise part of the
/// contract — openWakeWord's training did it, so inference must too, and getting it wrong
/// produces a model that runs perfectly and never fires.
internal sealed class WakeWord : IDisposable
{
    /// Mel frames the embedding model expects, and how far the window moves each time.
    private const int MelWindow = 76;
    private const int MelStride = 8;

    /// Embeddings the classifier expects, and how wide each one is.
    private const int EmbeddingWindow = 16;
    private const int EmbeddingWidth = 96;

    /// Audio per melspectrogram call. 1,280 samples is 80 ms at 16 kHz, which is the chunk
    /// openWakeWord itself uses — smaller costs more calls for no more resolution.
    private const int ChunkSamples = 1280;

    private readonly InferenceSession _mel;
    private readonly InferenceSession _embed;
    private readonly InferenceSession _wake;
    private readonly string _wakeInput;

    private readonly List<float> _audio = new(ChunkSamples * 2);
    private readonly List<float[]> _mels = new(MelWindow * 2);
    private readonly List<float[]> _embeddings = new(EmbeddingWindow * 2);

    private bool _disposed;

    public string Phrase { get; }

    /// The last score, for the log and the diagnostics panel. A wake word that never fires
    /// is indistinguishable from a broken microphone unless somebody can see it *nearly*
    /// firing, which is the whole reason this is exposed rather than kept inside.
    public float LastScore { get; private set; }

    public WakeWord(string phrase, string melPath, string embedPath, string wakePath)
    {
        Phrase = phrase;

        var options = new SessionOptions
        {
            // One thread each. Three tiny models running eighty times a second do not want
            // a thread pool; they want to not disturb Whisper, which is the expensive one.
            InterOpNumThreads = 1,
            IntraOpNumThreads = 1,
            GraphOptimizationLevel = GraphOptimizationLevel.ORT_ENABLE_ALL
        };

        _mel = new InferenceSession(melPath, options);
        _embed = new InferenceSession(embedPath, options);
        _wake = new InferenceSession(wakePath, options);

        // Read rather than assumed: the classifiers do not all name their input the same.
        _wakeInput = _wake.InputMetadata.Keys.First();
    }

    /// Feeds audio in and answers whether the phrase was just said.
    ///
    /// Expects the same 16 kHz mono float samples the voice detector reads, so it can be fed
    /// from the identical stream rather than opening or resampling anything.
    public bool Heard(ReadOnlySpan<float> samples, float threshold)
    {
        if (_disposed) return false;

        foreach (var sample in samples) _audio.Add(sample);

        var fired = false;

        while (_audio.Count >= ChunkSamples)
        {
            var chunk = _audio.GetRange(0, ChunkSamples).ToArray();
            _audio.RemoveRange(0, ChunkSamples);

            if (Score(chunk) is { } score)
            {
                LastScore = score;
                if (score >= threshold) fired = true;
            }
        }

        /* Cleared on a hit, and this matters more than it looks. The classifier reads a
           sliding window of embeddings, so the phrase stays inside that window for several
           more chunks after it is recognised — without this she would wake, and wake again,
           and wake again, from one *"Hey Octavia"*. */
        if (fired)
        {
            _mels.Clear();
            _embeddings.Clear();
        }

        return fired;
    }

    private float? Score(float[] chunk)
    {
        // ---- 1. audio → mel frames ------------------------------------------------
        var audio = new DenseTensor<float>(chunk, [1, chunk.Length]);

        using (var melled = _mel.Run([NamedOnnxValue.CreateFromTensor(_mel.InputMetadata.Keys.First(), audio)]))
        {
            var output = melled.First().AsTensor<float>();

            // (1, 1, frames, 32) — and the transform below is part of openWakeWord's
            // contract, not a tuning knob. Training applied it; inference must match.
            var frames = output.Dimensions[^2];
            var bins = output.Dimensions[^1];

            for (var f = 0; f < frames; f++)
            {
                var frame = new float[bins];
                for (var b = 0; b < bins; b++) frame[b] = output[0, 0, f, b] / 10f + 2f;
                _mels.Add(frame);
            }
        }

        // ---- 2. mel window → embedding --------------------------------------------
        while (_mels.Count >= MelWindow)
        {
            var window = new DenseTensor<float>([1, MelWindow, _mels[0].Length, 1]);

            for (var f = 0; f < MelWindow; f++)
                for (var b = 0; b < _mels[f].Length; b++)
                    window[0, f, b, 0] = _mels[f][b];

            using var embedded = _embed.Run(
                [NamedOnnxValue.CreateFromTensor(_embed.InputMetadata.Keys.First(), window)]);

            var vector = embedded.First().AsTensor<float>();
            var embedding = new float[EmbeddingWidth];
            for (var i = 0; i < EmbeddingWidth; i++) embedding[i] = vector.GetValue(i);

            _embeddings.Add(embedding);
            _mels.RemoveRange(0, MelStride);

            // Unbounded growth is the failure this prevents: she may listen for hours.
            if (_embeddings.Count > EmbeddingWindow * 2)
                _embeddings.RemoveRange(0, _embeddings.Count - EmbeddingWindow);
        }

        // ---- 3. embeddings → is that the phrase? ----------------------------------
        if (_embeddings.Count < EmbeddingWindow) return null;

        var recent = new DenseTensor<float>([1, EmbeddingWindow, EmbeddingWidth]);
        var first = _embeddings.Count - EmbeddingWindow;

        for (var e = 0; e < EmbeddingWindow; e++)
            for (var i = 0; i < EmbeddingWidth; i++)
                recent[0, e, i] = _embeddings[first + e][i];

        using var judged = _wake.Run([NamedOnnxValue.CreateFromTensor(_wakeInput, recent)]);
        return judged.First().AsTensor<float>().GetValue(0);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        _mel.Dispose();
        _embed.Dispose();
        _wake.Dispose();
    }
}
```
