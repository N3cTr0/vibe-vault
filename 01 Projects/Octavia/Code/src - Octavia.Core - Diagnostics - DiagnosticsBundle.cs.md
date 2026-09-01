---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Diagnostics\DiagnosticsBundle.cs
---

# src\Octavia.Core\Diagnostics\DiagnosticsBundle.cs

```csharp
using System.IO.Compression;
using System.Text;
using System.Text.Json.Nodes;
using Octavia.Core;

namespace Octavia.Diagnostics;

/// One zip a non-technical person can email back, holding everything needed to diagnose
/// a fault on a machine nobody here can touch.
///
/// It is also the file that leaves the building, so what it contains is a deliberate
/// decision rather than "whatever was lying around": the log holds transcripts of things
/// said in the room, and README.txt says so before anyone sends it.
internal static partial class DiagnosticsBundle
{
    /// Words that make a setting's value too dangerous to copy out. The API key lives in
    /// a DPAPI-sealed file that is not in this bundle at all — this is here so the
    /// guarantee survives someone adding a setting later.
    private static readonly string[] Sensitive = ["key", "token", "secret", "password"];

    public static string SuggestedFileName() =>
        $"octavia-diagnostics-{DateTime.Now:yyyy-MM-dd-HHmm}.zip";

    public static async Task WriteAsync(
        string path, OctaviaConfig config, HostSnapshot host, CancellationToken cancel = default)
    {
        var checks = await SelfTest.RunAsync(config, host, cancel);
        var facts = SystemReport.Gather(config, host);

        using var zip = new ZipArchive(File.Create(path), ZipArchiveMode.Create);

        await WriteTextAsync(zip, "README.txt", Readme(), cancel);
        await WriteTextAsync(zip, "report.txt", Report(facts, checks), cancel);
        await WriteTextAsync(zip, "config.json", RedactedConfig(), cancel);

        foreach (var file in Log.Files())
        {
            try { await WriteTextAsync(zip, $"logs/{Path.GetFileName(file)}", ReadShared(file), cancel); }
            catch (Exception ex) { Log.Warn($"could not include {Path.GetFileName(file)} in the bundle: {ex.Message}"); }
        }

        Log.Write($"diagnostics bundle written to {path}");
    }

    private static string Readme() =>
        $"""
         Octavia diagnostics — {DateTime.Now:MM/dd/yyyy HH:mm}
         =====================================================

         WHAT IS IN THIS FILE

           README.txt    this note
           report.txt    versions, hardware, audio devices and the result of her self-test
           config.json   her settings, with anything named like a key or token removed
           logs/         octavia.log and its rolled predecessors

         WHAT IT DOES NOT CONTAIN

           Your API key. It is sealed to your Windows account with DPAPI, stored outside
           this bundle, and is never written to the log or sent to the face.

         PLEASE READ BEFORE SENDING

           The log records what Octavia heard and said. If you have been talking to her,
           it contains transcripts of things said in the room, and file paths include your
           Windows account name.

           Open logs\octavia.log in Notepad and read it first. Delete anything you would
           rather not share — the bundle is still useful without it.

         Version {SystemReport.Version}
         """;

    private static string Report(IReadOnlyList<Fact> facts, IReadOnlyList<Check> checks)
    {
        var text = new StringBuilder();
        text.AppendLine($"Octavia diagnostics report — {DateTime.Now:MM/dd/yyyy HH:mm:ss}");
        text.AppendLine();

        text.AppendLine("SELF-TEST");
        text.AppendLine("---------");
        foreach (var check in checks)
        {
            text.AppendLine($"[{(check.Ok ? " ok " : "FAIL")}] {check.Name}: {check.Detail}");
            if (!check.Ok && check.Fix is not null) text.AppendLine($"         -> {check.Fix}");
        }

        text.AppendLine();
        text.AppendLine("SYSTEM");
        text.AppendLine("------");
        var width = facts.Max(f => f.Name.Length);
        foreach (var fact in facts) text.AppendLine($"{fact.Name.PadRight(width)}  {fact.Value}");

        text.AppendLine();
        text.AppendLine("RECENT LOG");
        text.AppendLine("----------");
        foreach (var line in Log.Tail(80)) text.AppendLine(line);

        return text.ToString();
    }

    /// A copy of the settings with anything key-shaped stripped, so the bundle can be
    /// sent without reading every line of it first.
    private static string RedactedConfig()
    {
        try
        {
            if (!File.Exists(Paths.ConfigFile)) return "// no config.json on this machine";

            var root = JsonNode.Parse(ReadShared(Paths.ConfigFile))?.AsObject();
            if (root is null) return "// config.json could not be parsed";

            Redact(root);

            // Relaxed escaping because a person reads this file: the default encoder
            // turns "Ctrl+Alt+O" into "Ctrl+Alt+O", which looks like damage.
            return root.ToJsonString(new System.Text.Json.JsonSerializerOptions
            {
                WriteIndented = true,
                Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            });
        }
        catch (Exception ex)
        {
            return $"// config.json could not be read: {ex.Message}";
        }
    }

    private static void Redact(JsonObject node)
    {
        foreach (var (name, value) in node.ToList())
        {
            if (value is JsonObject nested) Redact(nested);
            else if (IsSensitive(name) && value?.GetValueKind() == System.Text.Json.JsonValueKind.String)
                node[name] = "[removed]";
        }
    }

    /// A plain substring match blanked `Hotkey` and `MaxTokens`, which are two of the
    /// most useful lines in the file. Match whole words of the name instead, and only
    /// ever redact a *string* — a secret is never a number or a boolean.
    private static bool IsSensitive(string name) =>
        Words().Split(name)
               .Where(word => word.Length > 0)
               .Select(word => word.TrimEnd('s', 'S'))
               .Any(word => Sensitive.Contains(word, StringComparer.OrdinalIgnoreCase));

    [System.Text.RegularExpressions.GeneratedRegex(@"(?<!^)(?=[A-Z])|[^A-Za-z0-9]+")]
    private static partial System.Text.RegularExpressions.Regex Words();

    /// She may be writing to the log at this moment; a bundle must never fail because
    /// of that.
    private static string ReadShared(string path)
    {
        using var stream = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
        using var reader = new StreamReader(stream);
        return reader.ReadToEnd();
    }

    private static async Task WriteTextAsync(ZipArchive zip, string name, string content, CancellationToken cancel)
    {
        var entry = zip.CreateEntry(name, CompressionLevel.Optimal);
        await using var stream = entry.Open();
        await using var writer = new StreamWriter(stream, new UTF8Encoding(false));
        await writer.WriteAsync(content.AsMemory(), cancel);
    }
}
```
