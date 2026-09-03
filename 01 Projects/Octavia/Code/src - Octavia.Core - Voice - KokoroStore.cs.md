---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Voice\KokoroStore.cs
---

# src\Octavia.Core\Voice\KokoroStore.cs

```csharp
using System.Diagnostics;
using Octavia.Core;

namespace Octavia.Voice;

/// Fetches her voice, once, the same way the ears fetch Whisper: on first use, into her
/// data folder, with progress on her face.
///
/// **There is exactly one voice, and this file is where that is written down.** She was
/// auditioned against twenty-one others in Stage 16 and the owner picked this one by ear;
/// everything about it below is a constant rather than a setting, because a choice already
/// made is not a preference. See `ROADMAP.md` Stage 16.
internal static class KokoroStore
{
    /// The English catalogue is in `v1_0`. `v1_1` is newer and lists 103 speakers against
    /// 53, which makes it look like the obvious download — a hundred of them are Chinese.
    private const string Archive =
        "https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-multi-lang-v1_0.tar.bz2";

    private const string Folder = "kokoro-multi-lang-v1_0";

    /// Her voice. `Speaker` is this voice's index in `voices.bin` — it belongs to the model
    /// file, not to us, and changing it changes who she is.
    public const string Voice = "af_heart";
    public const int Speaker = 3;

    /// Kokoro's output rate. Read from the running engine rather than trusted (it prints it
    /// on startup); this is only the value to assume before it has spoken.
    public const int SampleRate = 24000;

    public static string Root { get; } =
        Directory.CreateDirectory(Path.Combine(Paths.DataDir, "voices")).FullName;

    public static string ModelDir => Path.Combine(Root, Folder);

    public static bool HasVoice =>
        File.Exists(Path.Combine(ModelDir, "model.onnx")) &&
        File.Exists(Path.Combine(ModelDir, "voices.bin"));

    /// The engine is a separate executable — see `Octavia.Kokoro`. Two layouts exist, the
    /// same two `LocalServer.FindServer` handles: shipped, where everything is side by side
    /// in `dist`, and a working tree, where every project has its own `bin` and the two are
    /// cousins. A path that only works after `dotnet publish` is a path nobody exercises
    /// while writing the code that depends on it.
    public static string? EnginePath()
    {
        const string exe = "octavia-kokoro.exe";

        var beside = Path.Combine(AppContext.BaseDirectory, exe);
        if (File.Exists(beside)) return beside;

        var here = new DirectoryInfo(AppContext.BaseDirectory);

        while (here is not null)
        {
            var bin = Path.Combine(here.FullName, "src", "Octavia.Kokoro", "bin");

            if (Directory.Exists(bin))
            {
                // Newest wins: a Debug and a Release build both exist in a working tree, and
                // the one just built is the one being tested.
                var found = Directory.EnumerateFiles(bin, exe, SearchOption.AllDirectories)
                                     .OrderByDescending(File.GetLastWriteTimeUtc)
                                     .FirstOrDefault();
                if (found is not null) return found;
            }

            here = here.Parent;
        }

        return null;
    }

    public static async Task EnsureAsync(Action<string>? progress, CancellationToken cancel = default)
    {
        if (HasVoice) return;

        var archive = Path.Combine(Root, $"{Folder}.tar.bz2");

        if (!File.Exists(archive))
        {
            progress?.Invoke("Fetching her voice (about 350 MB, once)...");
            await DownloadAsync(Archive, archive, cancel);
        }

        progress?.Invoke("Unpacking her voice...");
        Unpack(archive);

        if (!HasVoice) throw new InvalidOperationException($"the voice unpacked but nothing is in {ModelDir}");

        // 350 MB that is never read again. Kept until the model is proven present, so an
        // interrupted unpack does not turn one download into two.
        try { File.Delete(archive); }
        catch (Exception ex) { Log.Debug($"could not remove {archive}: {ex.Message}"); }

        Log.Write($"her voice is installed at {ModelDir}");
    }

    /// Unpacked by Windows' own `tar`, which is bsdtar and reads bzip2.
    ///
    /// .NET has Deflate, GZip, ZLib and Brotli and nothing that opens a `.tar.bz2`, and the
    /// archive is what the model is published as. Taking a compression library into her for
    /// one call, made once per machine, is the worse trade — `tar.exe` has shipped in
    /// Windows since 1803.
    private static void Unpack(string archive)
    {
        var tar = Process.Start(new ProcessStartInfo("tar")
        {
            WorkingDirectory = Root,
            ArgumentList = { "-xjf", archive },
            UseShellExecute = false,
            RedirectStandardError = true,
            CreateNoWindow = true
        }) ?? throw new InvalidOperationException("tar would not start");

        var complaints = tar.StandardError.ReadToEnd();
        if (!tar.WaitForExit(300_000)) { tar.Kill(true); throw new TimeoutException("unpacking her voice took too long"); }
        if (tar.ExitCode != 0) throw new InvalidOperationException($"could not unpack her voice: {complaints.Trim()}");
    }

    /// Downloads beside the target and moves on success, so an interrupted fetch never
    /// leaves half an archive looking like a whole one.
    private static async Task DownloadAsync(string url, string path, CancellationToken cancel)
    {
        var partial = path + ".partial";

        using var http = new HttpClient { Timeout = TimeSpan.FromMinutes(30) };
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
