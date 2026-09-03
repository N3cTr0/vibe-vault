---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\Ears\WakeWordStore.cs
---

# src\Octavia.Core\Senses\Ears\WakeWordStore.cs

```csharp
using Octavia.Core;

namespace Octavia.Senses;

/// The three models a wake word needs, fetched once.
///
/// **openWakeWord is three models, not one**, and that is the thing to know before reading
/// `WakeWord`. Two of them are shared by every wake word there is — a melspectrogram front
/// end and a speech-embedding backbone — and only the third, the small classifier, knows
/// which phrase it is listening for. So adding a second wake word later costs about a
/// megabyte, not four.
///
/// **Downloaded rather than shipped**, exactly as Whisper and her voice are, and for the same
/// reason: a repository is not a good place to keep binaries that a release page already
/// versions. Unlike Whisper this is a few megabytes, so it happens once and quickly.
internal static class WakeWordStore
{
    /// Pinned. A wake word model is trained against a *specific* embedding backbone, and
    /// silently pairing a v0.5.1 classifier with some later feature extractor is the kind of
    /// mistake that does not throw — it just never fires, which is indistinguishable from a
    /// microphone that is not working.
    private const string Release =
        "https://github.com/dscripka/openWakeWord/releases/download/v0.5.1";

    public static string ModelsDir { get; } =
        Directory.CreateDirectory(Path.Combine(Paths.DataDir, "models", "wake")).FullName;

    public const string Melspectrogram = "melspectrogram.onnx";
    public const string Embedding = "embedding_model.onnx";

    /// The pretrained phrases openWakeWord publishes.
    ///
    /// **None of them is "Octavia"**, and that is the honest state of this: hers has to be
    /// trained, off this machine, and the training pipeline is pinned to 2022-era PyTorch and
    /// TensorFlow. These exist so the whole path can be *proved* with a real model before
    /// anybody spends ninety minutes in a Colab — and so that "it does not fire" can be told
    /// apart from "the plumbing is wrong".
    public static readonly IReadOnlyDictionary<string, string> Pretrained =
        new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
        {
            ["hey jarvis"] = "hey_jarvis_v0.1.onnx",
            ["alexa"] = "alexa_v0.1.onnx",
            ["hey mycroft"] = "hey_mycroft_v0.1.onnx",
            ["hey rhasspy"] = "hey_rhasspy_v0.1.onnx"
        };

    public static string PathFor(string file) => Path.Combine(ModelsDir, file);

    /// The classifier for a phrase: a pretrained one by name, or a file somebody has put in
    /// the folder themselves — which is how `hey_octavia.onnx` will arrive.
    public static string? ResolveWakeModel(string phrase)
    {
        if (string.IsNullOrWhiteSpace(phrase)) return null;

        if (Pretrained.TryGetValue(phrase.Trim(), out var known)) return known;

        // A phrase that is not one of theirs is assumed to be a file. `Hey Octavia` →
        // `hey_octavia.onnx`, which is what the Colab notebooks name their output.
        var file = phrase.Trim().ToLowerInvariant().Replace(' ', '_') + ".onnx";
        return File.Exists(PathFor(file)) ? file : null;
    }

    public static bool Has(string file) => File.Exists(PathFor(file));

    /// Fetches what is missing. Returns false rather than throwing: a wake word that cannot
    /// be downloaded must leave her listening the way she did before, not stop her working.
    public static async Task<bool> EnsureAsync(
        string wakeModel, Action<string>? progress = null, CancellationToken cancel = default)
    {
        foreach (var file in new[] { Melspectrogram, Embedding, wakeModel })
        {
            if (Has(file)) continue;

            /* A wake word somebody trained themselves is not on that release page, so there
               is nothing to fetch and nothing to apologise for — it is simply missing, and
               saying which file would be a great deal more useful than a download error. */
            if (!Pretrained.Values.Contains(file) &&
                file != Melspectrogram && file != Embedding)
            {
                Log.Warn($"wake word: {PathFor(file)} is not there, and is not one to download");
                return false;
            }

            if (!await FetchAsync(file, progress, cancel)) return false;
        }

        return true;
    }

    private static async Task<bool> FetchAsync(
        string file, Action<string>? progress, CancellationToken cancel)
    {
        var path = PathFor(file);
        var partial = path + ".partial";

        progress?.Invoke($"Fetching {file}. This happens once.");
        Log.Write($"wake word: downloading {file}");

        try
        {
            using var http = new HttpClient { Timeout = TimeSpan.FromMinutes(2) };
            await using (var source = await http.GetStreamAsync($"{Release}/{file}", cancel))
            await using (var target = File.Create(partial))
                await source.CopyToAsync(target, cancel);

            // Moved into place only once it is whole, so an interrupted download cannot be
            // mistaken for a model next time. The same rule Whisper's store follows.
            File.Move(partial, path, overwrite: true);
            Log.Write($"wake word: {file} ready ({new FileInfo(path).Length >> 10} KB)");
            return true;
        }
        catch (Exception ex)
        {
            Log.Warn($"wake word: could not fetch {file}: {ex.Message}");
            try { if (File.Exists(partial)) File.Delete(partial); } catch { /* nothing to do */ }
            return false;
        }
    }
}
```
