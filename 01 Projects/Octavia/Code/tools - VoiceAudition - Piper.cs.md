---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Piper.cs
---

# tools\VoiceAudition\Piper.cs

```csharp
using System.Diagnostics;

namespace VoiceAudition;

/// The incumbent engine, rendering voices it already knows how to fetch.
///
/// Worth doing first and worth doing properly: if one of these is simply nicer than Amy,
/// Stage 16 costs one line of config and no architecture at all. Ruling that in or out
/// before looking at a new engine is the roadmap's own instruction.
internal static class Piper
{
    private const string VoicesRoot = "https://huggingface.co/rhasspy/piper-voices/resolve/main";

    /// English female voices, including the three `high` models that were never on her
    /// shortlist. `high` is not a label - it is a larger model at the same sample rate, and
    /// it is the most likely place for the incumbent engine to have a better answer hiding.
    private static readonly (string Voice, string Note)[] Candidates =
    [
        ("en_US-amy-medium",                 "the incumbent - judge the rest against this"),
        ("en_GB-cori-high",                  "British, high"),
        ("en_US-ljspeech-high",              "American, high"),
        ("en_US-lessac-high",                "American, high"),
        ("en_US-lessac-medium",              "the same speaker at medium - what 'high' buys"),
        ("en_GB-jenny_dioco-medium",         "British"),
        ("en_GB-alba-medium",                "Scottish"),
        ("en_US-hfc_female-medium",          "American"),
        ("en_US-kristin-medium",             "American"),
        ("en_GB-southern_english_female-low", "British, low - included as the floor")
    ];

    public static async Task<IReadOnlyList<string>> RenderAsync(string script)
    {
        Console.WriteLine("Piper");
        Console.WriteLine("-----");

        if (!File.Exists(Paths.PiperExe))
        {
            Console.WriteLine($"  piper.exe is not at {Paths.PiperExe} - run her once so it downloads, then try again.");
            return [];
        }

        var made = new List<string>();

        foreach (var (voice, note) in Candidates)
        {
            var model = Path.Combine(Paths.Voices, $"{voice}.onnx");
            var config = model + ".json";

            try
            {
                if (!File.Exists(model) || !File.Exists(config))
                {
                    Console.WriteLine($"  fetching {voice}...");
                    var url = Url(voice);
                    await Download.ToAsync($"{url}.onnx", model);
                    await Download.ToAsync($"{url}.onnx.json", config);
                }

                var wav = Path.Combine(Paths.Auditions, $"Piper - {Pretty(voice)}.wav");
                Speak(model, config, script, wav);

                Console.WriteLine($"  ok   {Path.GetFileName(wav),-46} {note}");
                made.Add(wav);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"  FAIL {voice} - {ex.Message}");
            }
        }

        return made;
    }

    /// en_GB-jenny_dioco-medium -> en/en_GB/jenny_dioco/medium/en_GB-jenny_dioco-medium
    private static string Url(string voice)
    {
        var parts = voice.Split('-');
        var family = parts[0].Split('_')[0];
        return $"{VoicesRoot}/{family}/{parts[0]}/{string.Join('-', parts[1..^1])}/{parts[^1]}/{voice}";
    }

    private static void Speak(string model, string config, string script, string wav)
    {
        var start = new ProcessStartInfo(Paths.PiperExe)
        {
            // espeak-ng's data sits beside the executable; started anywhere else it
            // phonemises nothing and writes a silent file rather than failing.
            WorkingDirectory = Path.GetDirectoryName(Paths.PiperExe)!,
            RedirectStandardInput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true
        };

        start.ArgumentList.Add("--model");
        start.ArgumentList.Add(model);
        start.ArgumentList.Add("--config");
        start.ArgumentList.Add(config);
        start.ArgumentList.Add("--output_file");
        start.ArgumentList.Add(wav);

        using var piper = Process.Start(start) ?? throw new InvalidOperationException("piper would not start");
        piper.StandardInput.WriteLine(script.Replace('\n', ' '));
        piper.StandardInput.Close();

        var complaints = piper.StandardError.ReadToEnd();
        if (!piper.WaitForExit(120_000)) { piper.Kill(true); throw new TimeoutException("piper did not finish"); }

        // It exits 0 having written nothing when the model and config disagree, so the file
        // is the only honest evidence that it worked.
        if (!File.Exists(wav) || new FileInfo(wav).Length < 1024)
            throw new InvalidOperationException($"no audio came out. {complaints.Trim()}");
    }

    private static string Pretty(string voice)
    {
        var parts = voice.Split('-');
        var name = string.Join(' ', parts[1..^1]).Replace('_', ' ');
        name = string.Join(' ', name.Split(' ').Select(w => w.Length > 0 ? char.ToUpperInvariant(w[0]) + w[1..] : w));
        return $"{name} ({parts[0].Replace('_', '-')}, {parts[^1]})";
    }
}
```
