---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\DiagnosticsChecks.cs
---

# tools\EarsTest\DiagnosticsChecks.cs

```csharp
// The diagnostics are what stands in for a debugger on someone else's machine, so they
// are exactly the thing that must not be broken. A bundle that silently omits the log,
// or a rotation that never rotates, would only be discovered when it was too late to
// matter.
using System.IO.Compression;
using Octavia.Core;
using Octavia.Diagnostics;
using Octavia.Face;

internal static class DiagnosticsChecks
{
    public static async Task<int> RunAsync()
    {
        var failures = 0;
        var scratch = Directory.CreateDirectory(
            Path.Combine(Path.GetTempPath(), $"octavia-diagtest-{Guid.NewGuid():N}")).FullName;

        var previousLog = Environment.GetEnvironmentVariable("OCTAVIA_LOG");
        var previousConfig = Environment.GetEnvironmentVariable("OCTAVIA_CONFIG");
        var previousMax = Log.MaxBytes;
        var previousThreshold = Log.Threshold;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        try
        {
            var logPath = Path.Combine(scratch, "octavia.log");
            Environment.SetEnvironmentVariable("OCTAVIA_LOG", logPath);
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", Path.Combine(scratch, "config.json"));

            // --- levels -------------------------------------------------
            Log.Threshold = LogLevel.Info;
            Log.Debug("this must not appear");
            Log.Write("plain info line");
            Log.Warn("a warning");
            Log.Error("an error");

            // Nothing is written to the base name any more; the day is spliced in, so
            // midnight rotates the log by being a different answer to "what is today".
            logPath = Log.Today;

            var written = File.ReadAllText(logPath);
            Check("debug is filtered out", !written.Contains("must not appear"), "debug line was written at info level");
            Check("info is written", written.Contains("plain info line"), "info line missing");
            Check("levels are labelled", written.Contains("ERROR") && written.Contains("warn"), written);

            Log.Threshold = LogLevel.Debug;
            Log.Debug("now visible");
            Check("threshold can be lowered", File.ReadAllText(logPath).Contains("now visible"), "debug still filtered");

            Check("tail remembers lines", Log.Tail(10).Any(l => l.Contains("an error")), "the error is not in the tail");

            // --- one file per day ---------------------------------------
            Check("the log is named for the day",
                  Path.GetFileName(logPath) == $"octavia-{DateTime.Now:yyyy-MM-dd}.log",
                  Path.GetFileName(logPath));

            // --- rotation within a day ----------------------------------
            Log.MaxBytes = 2000;
            for (var i = 0; i < 60; i++) Log.Write(new string('x', 80) + i);

            var files = Log.Files();
            Check("log rolled", files.Count > 1, $"{files.Count} file(s) after exceeding the cap");
            Check("current log is small", new FileInfo(logPath).Length < 8000,
                $"{new FileInfo(logPath).Length} bytes");
            Check("rolled file kept",
                  File.Exists(Path.Combine(scratch, $"octavia-{DateTime.Now:yyyy-MM-dd}.1.log")),
                  "the rolled file is missing");

            /* --- purging ------------------------------------------------

               Backdated files rather than a clock that has to be waited out, and **the file's
               own timestamp is what the purge reads** — which is what makes it quietly do the
               right thing with the `octavia.log` and `octavia.1.log` every version before this
               one left lying about. One of those is planted here on purpose. */
            var ancient = Path.Combine(scratch, "octavia-2020-01-01.log");
            var legacy = Path.Combine(scratch, "octavia.1.log");
            var recent = Path.Combine(scratch, $"octavia-{DateTime.Now.AddDays(-1):yyyy-MM-dd}.log");

            foreach (var (file, age) in new[] { (ancient, 400), (legacy, 400), (recent, 1) })
            {
                File.WriteAllText(file, "old\n");
                File.SetLastWriteTime(file, DateTime.Now.AddDays(-age));
            }

            Log.KeepDays = 14;
            Log.Purged = DateOnly.MinValue;   // as though the day had just turned over
            Log.Write("a line that triggers the purge");

            Check("a log older than the limit is deleted", !File.Exists(ancient), "it is still there");
            Check("...including one named the old way", !File.Exists(legacy), "it is still there");
            Check("a log inside the limit is kept", File.Exists(recent), "yesterday's was deleted");
            Check("...and today's is never touched", File.Exists(logPath), "today's log was deleted");

            /* Zero means keep everything, which is a real answer for somebody chasing a fault
               across a month. It must not be read as "keep nothing", which is the reading that
               deletes the evidence they were collecting. */
            File.WriteAllText(ancient, "old\n");
            File.SetLastWriteTime(ancient, DateTime.Now.AddDays(-400));

            Log.KeepDays = 0;
            Log.Purged = DateOnly.MinValue;
            Log.Write("another line");

            Check("zero days keeps everything", File.Exists(ancient), "it was purged anyway");

            Log.KeepDays = 14;

            // --- self-test ----------------------------------------------
            var config = new OctaviaConfig { Brain = "local", Recognizer = "whisper" };
            var host = new HostSnapshot("test brain", true, "not started", "Test Voice", false, true,
                new FaceStatus(Page: true, SocketBound: true, Port: 8848, SocketFaces: 1));

            var checks = await SelfTest.RunAsync(config, host);
            Check("self-test returns checks", checks.Count >= 9, $"{checks.Count} checks");
            Check("every check is named", checks.All(c => !string.IsNullOrWhiteSpace(c.Name)), "a check has no name");
            Check("every failure names a fix", checks.All(c => c.Ok || !string.IsNullOrWhiteSpace(c.Fix)),
                string.Join(", ", checks.Where(c => !c.Ok && string.IsNullOrWhiteSpace(c.Fix)).Select(c => c.Name)));
            Check("transport check reads the snapshot",
                checks.Any(c => c.Name == "Face transport" && c.Ok && c.Detail.Contains("8848")),
                checks.FirstOrDefault(c => c.Name == "Face transport")?.Detail ?? "missing");

            // A bundle taken with --diagnostics while she is stopped: the machine still
            // has everything to say, the session has nothing.
            var stopped = await SelfTest.RunAsync(config, HostSnapshot.Stopped);
            Check("stopped bundle drops the transport check",
                stopped.All(c => c.Name != "Face transport"), "it claimed the socket had not bound");
            // "Microphone" was the other half of this until Stage 15 item 3 removed the
            // device it asked about. The point stands and is now carried by the checks that
            // are still about *her machine* rather than about a room's hardware.
            Check("stopped bundle still checks the machine",
                stopped.Any(c => c.Name == "Speech model") && stopped.Any(c => c.Name == "Data folder"),
                "machine checks were skipped too");

            // --- the bundle ---------------------------------------------
            // Hotkey and MaxTokens are here because a plain substring match blanked both,
            // and they are two of the most useful lines a fault report can carry.
            File.WriteAllText(Paths.ConfigFile,
                """
                {
                  "Brain": "local",
                  "Hotkey": "Ctrl+Alt+O",
                  "MaxTokens": 1024,
                  "SomeApiKey": "sk-ant-secret-value",
                  "Nested": { "Token": "abc123" }
                }
                """);

            var bundlePath = Path.Combine(scratch, "bundle.zip");
            await DiagnosticsBundle.WriteAsync(bundlePath, config, host);

            using var zip = ZipFile.OpenRead(bundlePath);
            var names = zip.Entries.Select(e => e.FullName).ToList();

            Check("bundle has a readme", names.Contains("README.txt"), string.Join(", ", names));
            Check("bundle has the report", names.Contains("report.txt"), string.Join(", ", names));
            Check("bundle has the settings", names.Contains("config.json"), string.Join(", ", names));
            Check("bundle has the logs", names.Any(n => n.StartsWith("logs/")), string.Join(", ", names));
            Check("bundle includes rolled logs", names.Count(n => n.StartsWith("logs/")) > 1,
                $"only {names.Count(n => n.StartsWith("logs/"))} log file(s)");

            var settings = Read(zip, "config.json");
            Check("key-shaped settings are redacted", !settings.Contains("sk-ant-secret-value"), settings);
            Check("nested tokens are redacted too", !settings.Contains("abc123"), settings);
            Check("ordinary settings survive", settings.Contains("local"), settings);
            Check("the hotkey is not mistaken for a secret", settings.Contains("Ctrl+Alt+O"), settings);
            Check("MaxTokens is not mistaken for a secret", settings.Contains("1024"), settings);

            var readme = Read(zip, "README.txt");
            Check("readme warns about transcripts",
                readme.Contains("transcripts", StringComparison.OrdinalIgnoreCase), readme);
            Check("readme says the key is absent",
                readme.Contains("API key", StringComparison.OrdinalIgnoreCase), readme);

            var report = Read(zip, "report.txt");
            Check("report carries the self-test", report.Contains("SELF-TEST"), "no self-test section");
            Check("report carries the system facts", report.Contains("Octavia") && report.Contains("Windows"),
                "no system section");

            Check("no key file in the bundle", !names.Any(n => n.Contains("apikey")), string.Join(", ", names));
        }
        finally
        {
            Log.MaxBytes = previousMax;
            Log.Threshold = previousThreshold;
            Environment.SetEnvironmentVariable("OCTAVIA_LOG", previousLog);
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", previousConfig);
            try { Directory.Delete(scratch, recursive: true); } catch (IOException) { }
        }

        return failures;
    }

    private static string Read(ZipArchive zip, string name)
    {
        var entry = zip.GetEntry(name);
        if (entry is null) return string.Empty;
        using var reader = new StreamReader(entry.Open());
        return reader.ReadToEnd();
    }
}
```
