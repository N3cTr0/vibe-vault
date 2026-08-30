---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Senses\Ears\WhisperCompute.cs
---

# src\Octavia.App\Senses\Ears\WhisperCompute.cs

```csharp
using Octavia.Core;
using Whisper.net.LibraryLoader;

namespace Octavia.Senses;

/// Which processor Whisper runs on.
///
/// Whisper.net picks a native library from its own order — [Cuda, Cuda12, Vulkan,
/// CoreML, OpenVino, Cpu, CpuNoAvx] — so merely referencing Whisper.net.Runtime.Cuda
/// hands every transcription to whatever card is present, however slow. On a machine
/// whose CPU is the stronger half that is a large, silent regression, and it looks
/// like Whisper being slow rather than like a choice anyone made.
internal static class WhisperCompute
{
    private static bool _applied;

    /// True when the factory should be built with GPU enabled.
    public static bool UseGpu { get; private set; } = true;

    /// The library actually loaded, once one has been. Null before that.
    public static string? Loaded => RuntimeOptions.LoadedLibrary?.ToString();

    /// Must run before the first WhisperFactory exists: RuntimeOptions is read when
    /// the native library loads and ignored from then on. Applying twice is harmless
    /// and the second call is ignored, so the first preference wins for the process.
    public static void Apply(string? preference)
    {
        if (_applied) return;
        _applied = true;

        switch (preference?.Trim().ToLowerInvariant())
        {
            case "cpu":
                RuntimeOptions.RuntimeLibraryOrder = [RuntimeLibrary.Cpu, RuntimeLibrary.CpuNoAvx];
                UseGpu = false;
                break;

            case "gpu":
                RuntimeOptions.RuntimeLibraryOrder =
                    [RuntimeLibrary.Cuda, RuntimeLibrary.Vulkan, RuntimeLibrary.Cpu, RuntimeLibrary.CpuNoAvx];
                UseGpu = true;
                break;

            default:
                UseGpu = true;
                break;
        }

        Log.Write($"whisper compute: {preference ?? "auto"} (gpu {(UseGpu ? "allowed" : "off")})");
    }
}
```
