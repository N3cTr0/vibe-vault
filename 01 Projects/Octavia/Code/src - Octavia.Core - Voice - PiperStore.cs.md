---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\PiperStore.cs
---

# src\Octavia.Core\Voice\PiperStore.cs

```csharp
using System.IO.Compression;
using Octavia.Core;

namespace Octavia.Voice;

/// Fetches the neural voice engine and its models, the same way the ears fetch Whisper:
/// once, on first use, into her data folder, with progress on her face.
///
/// **This one downloads an executable**, which is a different thing from downloading a
/// model file and worth saying out loud. It comes from the Piper project's own GitHub
/// release, it is written into her data folder rather than anywhere on the PATH, and the
/// URL is in this file where it can be read. Nothing runs until the user has asked for
/// the neural voice.
internal static class PiperStore
{
    private const string Release =
        "https://github.com/rhasspy/piper/releases/download/2023.11.14-2/piper_windows_amd64.zip";

    private const string VoicesRoot = "https://huggingface.co/rhasspy/piper-voices/resolve/main";

    /// A short list rather than the whole catalogue: every one of these has been listened
    /// to, and a menu of four hundred voices is not a kindness.
    public static readonly IReadOnlyList<string> Catalogue =
    [
        "en_GB-jenny_dioco-medium",
        "en_GB-alba-medium",
        "en_US-amy-medium",
        "en_US-hfc_female-medium",
        "en_US-lessac-medium"
    ];

    public static string Root { get; } =
        Directory.CreateDirectory(Path.Combine(Paths.DataDir, "voices")).FullName;

    public static string EngineDir => Path.Combine(Root, "piper");
    public static string EnginePath => Path.Combine(EngineDir, "piper", "piper.exe");
    public static string ModelPath(string voice) => Path.Combine(Root, $"{voice}.onnx");
    public static string ConfigPath(string voice) => Path.Combine(Root, $"{voice}.onnx.json");

    public static bool HasEngine => File.Exists(EnginePath);
    public static bool HasVoice(string voice) => File.Exists(ModelPath(voice)) && File.Exists(ConfigPath(voice));

    /// Whatever has actually been downloaded, so the settings menu offers what works.
    public static IReadOnlyList<string> Downloaded()
    {
        try
        {
            return Directory.EnumerateFiles(Root, "*.onnx")
                            .Select(Path.GetFileNameWithoutExtension)
                            .Where(name => name is not null && HasVoice(name))
                            .OrderBy(name => name, StringComparer.OrdinalIgnoreCase)
                            .ToList()!;
        }
        catch (Exception ex)
        {
            Log.Warn($"could not list {Root}: {ex.Message}");
            return [];
        }
    }

    public static async Task EnsureAsync(string voice, Action<string>? progress, CancellationToken cancel = default)
    {
        if (!HasEngine)
        {
            progress?.Invoke("Fetching the neural speech engine (about 22 MB)...");
            var zip = Path.Combine(Root, "piper.zip");
            await DownloadAsync(Release, zip, cancel);

            Directory.CreateDirectory(EngineDir);
            ZipFile.ExtractToDirectory(zip, EngineDir, overwriteFiles: true);
            File.Delete(zip);

            if (!HasEngine) throw new InvalidOperationException("the engine unpacked but piper.exe is not where it should be");
            Log.Write($"neural speech engine installed at {EngineDir}");
        }

        if (HasVoice(voice)) return;

        // en_GB-jenny_dioco-medium -> en/en_GB/jenny_dioco/medium/<full>.onnx
        var parts = voice.Split('-');
        if (parts.Length < 3) throw new ArgumentException($"'{voice}' is not a Piper voice name.");
        var family = parts[0].Split('_')[0];
        var folder = $"{VoicesRoot}/{family}/{parts[0]}/{string.Join('-', parts[1..^1])}/{parts[^1]}/{voice}";

        progress?.Invoke($"Fetching the voice '{Pretty(voice)}' (about 60 MB, once)...");
        await DownloadAsync($"{folder}.onnx", ModelPath(voice), cancel);
        await DownloadAsync($"{folder}.onnx.json", ConfigPath(voice), cancel);
        Log.Write($"neural voice '{voice}' downloaded");
    }

    /// "en_GB-jenny_dioco-medium" is a filename; "Jenny (en GB, medium)" is a menu entry.
    public static string Pretty(string voice)
    {
        var parts = voice.Split('-');
        if (parts.Length < 3) return voice;

        var name = string.Join(' ', parts[1..^1]).Replace('_', ' ');
        name = string.Join(' ', name.Split(' ').Select(w => w.Length > 0 ? char.ToUpperInvariant(w[0]) + w[1..] : w));
        return $"{name} ({parts[0].Replace('_', '-')}, {parts[^1]})";
    }

    /// Downloads beside the target and moves on success, so an interrupted fetch never
    /// leaves a half a model looking like a whole one.
    private static async Task DownloadAsync(string url, string path, CancellationToken cancel)
    {
        var partial = path + ".partial";

        using var http = new HttpClient { Timeout = TimeSpan.FromMinutes(10) };
        using var response = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, cancel);
        response.EnsureSuccessStatusCode();

        await using (var source = await response.Content.ReadAsStreamAsync(cancel))
        await using (var target = File.Create(partial))
        {
            await source.CopyToAsync(target, cancel);
        }

        File.Move(partial, path, overwrite: true);
    }
}
```
