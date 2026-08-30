---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\ConfigChecks.cs
---

# tools\EarsTest\ConfigChecks.cs

```csharp
// Which brain she starts with is decided here, and getting it wrong is silent —
// she just answers in the wrong voice from the wrong model. These checks pin the
// precedence order and the rule that a profile overlay never reaches the file.
using System.Text.Json;
using System.Text.Json.Nodes;
using Octavia.Core;

internal static class ConfigChecks
{
    public static int Run()
    {
        var failures = 0;
        var file = Path.Combine(Path.GetTempPath(), $"octavia-configtest-{Guid.NewGuid():N}.json");
        var previousConfig = Environment.GetEnvironmentVariable("OCTAVIA_CONFIG");
        var previousProfile = Environment.GetEnvironmentVariable("OCTAVIA_PROFILE");

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        try
        {
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", file);
            Environment.SetEnvironmentVariable("OCTAVIA_PROFILE", null);
            WriteBase(file);

            var fromFile = OctaviaConfig.Load();
            Check("file profile applies", fromFile.Brain == "claude", $"brain={fromFile.Brain}");

            var fromArgument = OctaviaConfig.Load("dev");
            Check("argument beats the file", fromArgument.Brain == "local", $"brain={fromArgument.Brain}");
            Check("argument switches the ears", fromArgument.WhisperModel == "small.en",
                $"whisper={fromArgument.WhisperModel}");

            Environment.SetEnvironmentVariable("OCTAVIA_PROFILE", "dev");
            var fromEnvironment = OctaviaConfig.Load();
            Check("environment beats the file", fromEnvironment.Brain == "local", $"brain={fromEnvironment.Brain}");

            Environment.SetEnvironmentVariable("OCTAVIA_PROFILE", "live");
            var argumentOverEnvironment = OctaviaConfig.Load("dev");
            Check("argument beats the environment", argumentOverEnvironment.Brain == "local",
                $"brain={argumentOverEnvironment.Brain}");
            Environment.SetEnvironmentVariable("OCTAVIA_PROFILE", null);

            var unknown = OctaviaConfig.Load("nonsense");
            Check("unknown profile falls back", unknown.Brain == "claude", $"brain={unknown.Brain}");

            // The regression that started this: saving while a profile was applied
            // used to bake the overlay into the base and every later run inherited it.
            var running = OctaviaConfig.Load("dev");
            running.VoiceName = "Test Voice";
            running.Save();

            var saved = JsonNode.Parse(File.ReadAllText(file))!.AsObject();
            Check("save keeps the base brain", (string?)saved["Brain"] == "claude",
                $"Brain={saved["Brain"]}");
            Check("save keeps the base whisper model", (string?)saved["WhisperModel"] == "large-v3-turbo",
                $"WhisperModel={saved["WhisperModel"]}");
            Check("save keeps the file's own profile", (string?)saved["Profile"] == "live",
                $"Profile={saved["Profile"]}");
            Check("save still records the voice", (string?)saved["VoiceName"] == "Test Voice",
                $"VoiceName={saved["VoiceName"]}");
            Check("save keeps both profiles", saved["Profiles"]?.AsObject().Count == 2,
                $"Profiles={saved["Profiles"]}");

            var reloaded = OctaviaConfig.Load("dev");
            Check("reload is still the dev brain", reloaded.Brain == "local", $"brain={reloaded.Brain}");

            // Carrying runtime changes back used to be a hand-kept list of properties,
            // which was wrong the moment a setting was added: the settings menu appeared
            // to work and silently persisted nothing.
            var settings = OctaviaConfig.Load("dev");
            settings.AvatarFile = "chosen.vrm";
            settings.RoomHour = 21;
            settings.Save();

            var afterSettings = JsonNode.Parse(File.ReadAllText(file))!.AsObject();
            Check("a new setting persists", (string?)afterSettings["AvatarFile"] == "chosen.vrm",
                $"AvatarFile={afterSettings["AvatarFile"]}");
            Check("a numeric setting persists", (int?)afterSettings["RoomHour"] == 21,
                $"RoomHour={afterSettings["RoomHour"]}");
            Check("the overlay still did not leak", (string?)afterSettings["Brain"] == "claude",
                $"Brain={afterSettings["Brain"]}");

            // Two saves in a row: the second must not undo the first.
            settings.VoiceName = "Second Voice";
            settings.Save();
            var afterTwice = JsonNode.Parse(File.ReadAllText(file))!.AsObject();
            Check("a second save keeps the first", (string?)afterTwice["AvatarFile"] == "chosen.vrm",
                $"AvatarFile={afterTwice["AvatarFile"]}");
            Check("a second save records its own change", (string?)afterTwice["VoiceName"] == "Second Voice",
                $"VoiceName={afterTwice["VoiceName"]}");
        }
        finally
        {
            Environment.SetEnvironmentVariable("OCTAVIA_CONFIG", previousConfig);
            Environment.SetEnvironmentVariable("OCTAVIA_PROFILE", previousProfile);
            try { File.Delete(file); } catch (IOException) { }
        }

        return failures;
    }

    private static void WriteBase(string path)
    {
        var config = new JsonObject
        {
            ["Profile"] = "live",
            ["Profiles"] = new JsonObject
            {
                ["dev"] = new JsonObject
                {
                    ["Brain"] = "local",
                    ["LocalModel"] = "llama3.2:3b",
                    ["WhisperModel"] = "small.en"
                },
                ["live"] = new JsonObject
                {
                    ["Brain"] = "claude",
                    ["WhisperModel"] = "large-v3-turbo"
                }
            },
            ["Brain"] = "claude",
            ["WhisperModel"] = "large-v3-turbo",
            ["VoiceName"] = "Original Voice"
        };

        File.WriteAllText(path, config.ToJsonString(new JsonSerializerOptions { WriteIndented = true }));
    }
}
```
