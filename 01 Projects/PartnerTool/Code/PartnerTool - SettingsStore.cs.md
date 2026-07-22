---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SettingsStore.cs
---

# PartnerTool\SettingsStore.cs

```csharp
using System.IO;
using System.Text.Json;

namespace PartnerTool;

/// <summary>User-tweakable settings, persisted to settings.json next to the exe.</summary>
public class AppSettings
{
    /// <summary>
    /// Logs in C:\PCI\Logs older than this many days are deleted at startup (the folder is
    /// ACL-hardened, so only the elevated app can tidy it). Clamped to 1–365; default 30.
    /// </summary>
    public int LogRetentionDays { get; set; } = 30;
}

/// <summary>
/// Loads/saves <see cref="AppSettings"/> as settings.json in the application folder
/// (the directory the exe runs from). Created with defaults on first run.
/// </summary>
public static class SettingsStore
{
    private static readonly string FilePath =
        Path.Combine(AppContext.BaseDirectory, "settings.json");

    public static AppSettings Current { get; private set; } = new();

    /// <summary>Raised after settings are saved so live UI can re-apply them.</summary>
    public static event Action? Changed;

    /// <summary>Load from disk; if the file is missing, create it with defaults.</summary>
    public static void Load()
    {
        try
        {
            if (File.Exists(FilePath))
            {
                Current = JsonSerializer.Deserialize<AppSettings>(File.ReadAllText(FilePath)) ?? new AppSettings();
            }
            else
            {
                Current = new AppSettings();
                Save();   // write the defaults so the file exists for editing
            }
        }
        catch { Current = new AppSettings(); }
    }

    /// <summary>Persist the current settings and notify listeners.</summary>
    public static void Save()
    {
        try
        {
            File.WriteAllText(FilePath,
                JsonSerializer.Serialize(Current, new JsonSerializerOptions { WriteIndented = true }));
        }
        catch { /* read-only location etc. — keep running with in-memory settings */ }
        Changed?.Invoke();
    }
}
```
