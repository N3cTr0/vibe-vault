---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Paths.cs
---

# tools\VoiceAudition\Paths.cs

```csharp
namespace VoiceAudition;

/// Her data folder, found by walking up rather than configured, so this tool works from
/// wherever `dotnet run` leaves the working directory.
internal static class Paths
{
    public static string Data { get; } = Find();

    public static string Auditions => Path.Combine(Data, "auditions");
    public static string Voices => Path.Combine(Data, "voices");
    public static string PiperExe => Path.Combine(Voices, "piper", "piper", "piper.exe");

    private static string Find()
    {
        var here = new DirectoryInfo(AppContext.BaseDirectory);

        while (here is not null)
        {
            var data = Path.Combine(here.FullName, "data");
            if (Directory.Exists(Path.Combine(here.FullName, "src")) && Directory.Exists(data)) return data;
            here = here.Parent;
        }

        throw new DirectoryNotFoundException("could not find Octavia's data folder above this executable");
    }
}
```
