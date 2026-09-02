---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Kokoro.cs
---

# tools\VoiceAudition\Kokoro.cs

```csharp
using System.Diagnostics;
using SherpaOnnx;

namespace VoiceAudition;

/// The challenger: Kokoro, 82M parameters, run through sherpa-onnx.
///
/// It is here rather than ElevenLabs because it keeps every constraint Stage 16 wrote down
/// before the search began - free, offline, private, out of process, raw PCM out (so her
/// mouth still reads off the waveform), and no per-character bill on someone who talks all
/// evening. A hosted voice would very likely sound better still, and it would also make an
/// outage into muteness. This is what "better" looks like without paying that.
internal static class Kokoro
{
    /// Two model directories, because they are not the same set of voices. `v1_0` carries
    /// the English catalogue; `v1_1` is overwhelmingly Chinese and contributes exactly three
    /// English women, who are newer recordings and worth hearing anyway.
    private static readonly (string Dir, int Sid, string Name, string Note)[] Candidates =
    [
        ("kokoro-multi-lang-v1_0",  3, "af_heart",     "American - the flagship voice"),
        ("kokoro-multi-lang-v1_0",  2, "af_bella",     "American - warm"),
        ("kokoro-multi-lang-v1_0",  6, "af_nicole",    "American - close, quiet"),
        ("kokoro-multi-lang-v1_0",  1, "af_aoede",     "American"),
        ("kokoro-multi-lang-v1_0",  5, "af_kore",      "American"),
        ("kokoro-multi-lang-v1_0",  9, "af_sarah",     "American"),
        ("kokoro-multi-lang-v1_0", 21, "bf_emma",      "British"),
        ("kokoro-multi-lang-v1_0", 22, "bf_isabella",  "British"),
        ("kokoro-multi-lang-v1_0", 23, "bf_lily",      "British"),
        ("kokoro-multi-lang-v1_1",  0, "af_maple",     "American - v1.1"),
        ("kokoro-multi-lang-v1_1",  1, "af_sol",       "American - v1.1"),
        ("kokoro-multi-lang-v1_1",  2, "bf_vale",      "British - v1.1")
    ];

    public static async Task<IReadOnlyList<string>> RenderAsync(string script)
    {
        Console.WriteLine();
        Console.WriteLine("Kokoro");
        Console.WriteLine("------");

        var made = new List<string>();

        foreach (var group in Candidates.GroupBy(c => c.Dir))
        {
            var root = Path.Combine(Paths.Voices, group.Key);

            try
            {
                await Archive.EnsureAsync(group.Key, root);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"  FAIL {group.Key} - {ex.Message}");
                continue;
            }

            using var tts = Open(root);

            foreach (var (_, sid, name, note) in group)
            {
                try
                {
                    var wav = Path.Combine(Paths.Auditions, $"Kokoro - {name}.wav");

                    /* **The constraint, measured rather than assumed.** Stage 16 called
                       real-time sentence-by-sentence synthesis "the single biggest thing that
                       makes her feel present rather than transactional". A model four times
                       the size of a Piper voice is exactly where that could quietly stop being
                       true, so the audition reports how long this sentence took against how
                       long it lasts. Under 1.0 means she can keep ahead of herself. */
                    var clock = Stopwatch.StartNew();
                    var audio = tts.Generate(script, speed: 1.0f, speakerId: sid);
                    clock.Stop();

                    if (!audio.SaveToWaveFile(wav)) throw new InvalidOperationException("nothing was written");

                    var seconds = audio.Samples.Length / (double)audio.SampleRate;
                    Console.WriteLine($"  ok   {Path.GetFileName(wav),-32} {note,-32} " +
                                      $"{clock.Elapsed.TotalSeconds:0.0}s for {seconds:0.0}s " +
                                      $"(RTF {clock.Elapsed.TotalSeconds / seconds:0.00})");
                    made.Add(wav);
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"  FAIL {name} - {ex.Message}");
                }
            }
        }

        return made;
    }

    private static OfflineTts Open(string root)
    {
        var config = new OfflineTtsConfig();
        config.Model.Kokoro.Model = Path.Combine(root, "model.onnx");
        config.Model.Kokoro.Voices = Path.Combine(root, "voices.bin");
        config.Model.Kokoro.Tokens = Path.Combine(root, "tokens.txt");
        config.Model.Kokoro.DataDir = Path.Combine(root, "espeak-ng-data");

        // The multilingual models want the Chinese word-segmenter and both lexicons even
        // when the text is English; without them they load and then phonemise nothing.
        var dict = Path.Combine(root, "dict");
        if (Directory.Exists(dict)) config.Model.Kokoro.DictDir = dict;

        var lexicons = new[] { "lexicon-us-en.txt", "lexicon-zh.txt" }
            .Select(f => Path.Combine(root, f))
            .Where(File.Exists);
        config.Model.Kokoro.Lexicon = string.Join(',', lexicons);

        config.Model.NumThreads = 4;
        config.Model.Provider = "cpu";
        config.Model.Debug = 0;

        return new OfflineTts(config);
    }
}
```
