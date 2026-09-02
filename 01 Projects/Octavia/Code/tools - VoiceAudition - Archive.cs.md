---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Archive.cs
---

# tools\VoiceAudition\Archive.cs

```csharp
using System.Diagnostics;

namespace VoiceAudition;

internal static class Archive
{
    private const string Models = "https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models";

    /// Fetches and unpacks a sherpa-onnx model archive, once.
    ///
    /// Unpacked by Windows' own `tar`, which is bsdtar and reads bzip2 - .NET has Deflate,
    /// GZip and Brotli and nothing that opens a `.tar.bz2`. Pulling in a compression library
    /// for a research tool that runs twice would be the wrong trade.
    public static async Task EnsureAsync(string name, string root)
    {
        if (File.Exists(Path.Combine(root, "model.onnx"))) return;

        var archive = Path.Combine(Paths.Voices, $"{name}.tar.bz2");

        if (!File.Exists(archive))
        {
            Console.WriteLine($"  fetching {name} (about 350 MB, once)...");
            await Download.ToAsync($"{Models}/{name}.tar.bz2", archive);
        }

        Console.WriteLine($"  unpacking {name}...");
        var tar = Process.Start(new ProcessStartInfo("tar")
        {
            WorkingDirectory = Paths.Voices,
            ArgumentList = { "-xjf", archive },
            UseShellExecute = false
        }) ?? throw new InvalidOperationException("tar would not start");

        await tar.WaitForExitAsync();

        if (!File.Exists(Path.Combine(root, "model.onnx")))
            throw new InvalidOperationException($"{name} unpacked but model.onnx is not in {root}");
    }
}
```
