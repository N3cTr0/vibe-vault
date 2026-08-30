---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Ears\WhisperModelStore.cs
---

# src\Octavia.App\Senses\Ears\WhisperModelStore.cs

```csharp
using Octavia.Core;
using Whisper.net.Ggml;

namespace Octavia.Senses;

internal static class WhisperModelStore
{
    public static string ModelsDir { get; } =
        Directory.CreateDirectory(Path.Combine(Paths.DataDir, "models")).FullName;

    public static string SileroPath =>
        Path.Combine(AppContext.BaseDirectory, "Assets", "models", "silero_vad.onnx");

    public static string PathFor(string model) => Path.Combine(ModelsDir, $"ggml-{model}.bin");

    public static GgmlType TypeFor(string model) => model switch
    {
        "tiny" => GgmlType.Tiny,
        "tiny.en" => GgmlType.TinyEn,
        "base" => GgmlType.Base,
        "base.en" => GgmlType.BaseEn,
        "small" => GgmlType.Small,
        "small.en" => GgmlType.SmallEn,
        "medium" => GgmlType.Medium,
        "medium.en" => GgmlType.MediumEn,
        "large-v2" => GgmlType.LargeV2,
        "large-v3" => GgmlType.LargeV3,
        "large-v3-turbo" => GgmlType.LargeV3Turbo,
        _ => throw new ArgumentException($"Unknown Whisper model '{model}'.")
    };

    /// Downloads the ggml file on first use. The big ones take a while, so progress
    /// goes back to the caller for the face to show.
    public static async Task<string> EnsureAsync(
        string model, Action<string>? progress = null, CancellationToken cancel = default)
    {
        var path = PathFor(model);
        if (File.Exists(path)) return path;

        var partial = path + ".partial";
        progress?.Invoke($"Downloading Whisper model {model}. This happens once.");
        Log.Write($"downloading whisper model {model}");

        try
        {
            await using var source = await WhisperGgmlDownloader.Default
                .GetGgmlModelAsync(TypeFor(model), cancellationToken: cancel);
            await using (var target = File.Create(partial))
            {
                var buffer = new byte[1 << 20];
                long total = 0, lastReport = 0;
                int read;
                while ((read = await source.ReadAsync(buffer, cancel)) > 0)
                {
                    await target.WriteAsync(buffer.AsMemory(0, read), cancel);
                    total += read;
                    if (total - lastReport >= 200 << 20)
                    {
                        lastReport = total;
                        progress?.Invoke($"Downloading {model}: {total >> 20} MB so far.");
                    }
                }
            }

            File.Move(partial, path);
            progress?.Invoke($"Model {model} ready.");
            Log.Write($"whisper model {model} downloaded to {path}");
            return path;
        }
        catch
        {
            if (File.Exists(partial)) File.Delete(partial);
            throw;
        }
    }
}
```
