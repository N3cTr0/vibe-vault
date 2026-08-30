---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Ears\WhisperTranscriber.cs
---

# src\Octavia.App\Senses\Ears\WhisperTranscriber.cs

```csharp
using System.Text;
using Octavia.Core;
using Whisper.net;

namespace Octavia.Senses;

internal sealed record Transcript(string Text, float Confidence);

internal sealed class WhisperTranscriber : IDisposable
{
    private readonly WhisperFactory _factory;
    private readonly WhisperProcessor _processor;
    private readonly SemaphoreSlim _oneAtATime = new(1, 1);

    public WhisperTranscriber(string modelPath, string language, string? compute = null, int threads = 0)
    {
        WhisperCompute.Apply(compute);
        _factory = WhisperFactory.FromPath(
            modelPath, new WhisperFactoryOptions { UseGpu = WhisperCompute.UseGpu });

        // Half the logical processors is physical-core count on every SMT machine, and
        // whisper.cpp gains nothing from the second thread on a core.
        var chosen = threads > 0 ? threads : Math.Max(2, Environment.ProcessorCount / 2);
        Log.Write($"whisper running on {WhisperCompute.Loaded ?? "an unreported library"}, {chosen} threads");

        var builder = _factory.CreateBuilder()
            .WithProbabilities()
            .WithThreads(chosen);

        _processor = (language == "auto"
            ? builder.WithLanguageDetection()
            : builder.WithLanguage(language)).Build();
    }

    /// samples: 16kHz mono float PCM of one utterance.
    public async Task<Transcript> TranscribeAsync(float[] samples, CancellationToken cancel = default)
    {
        await _oneAtATime.WaitAsync(cancel);
        try
        {
            var text = new StringBuilder();
            var probability = 0f;
            var segments = 0;

            await foreach (var segment in _processor.ProcessAsync(samples, cancel))
            {
                // Whisper's tells for audio it invented rather than heard: high
                // no-speech odds, or a bracketed non-speech tag like [BLANK_AUDIO].
                var trimmed = segment.Text.Trim();
                if ((segment.NoSpeechProbability > 0.6f && segment.Probability < 0.4f) ||
                    (trimmed.Length > 1 &&
                     trimmed[0] is '[' or '(' && trimmed[^1] is ']' or ')'))
                {
                    Log.Write($"dropped non-speech segment: {trimmed}");
                    continue;
                }

                text.Append(segment.Text);
                probability += segment.Probability;
                segments++;
            }

            return new Transcript(
                text.ToString().Trim(),
                segments > 0 ? probability / segments : 0f);
        }
        finally
        {
            _oneAtATime.Release();
        }
    }

    public void Dispose()
    {
        _processor.Dispose();
        _factory.Dispose();
        _oneAtATime.Dispose();
    }
}
```
