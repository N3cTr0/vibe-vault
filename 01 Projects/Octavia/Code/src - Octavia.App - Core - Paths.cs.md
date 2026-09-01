---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Core\Paths.cs
---

# src\Octavia.App\Core\Paths.cs

```csharp
namespace Octavia.Core;

internal static class Paths
{
    public static string DataDir { get; } = Directory.CreateDirectory(Resolve()).FullName;

    /// Run from the repo and everything she owns lives in the repo, so one folder is the
    /// whole of her and a machine move cannot separate her from her models. An installed
    /// copy falls back to %APPDATA%, which is the only writable choice under Program Files.
    /// OCTAVIA_DATA overrides both.
    private static string Resolve()
    {
        if (Environment.GetEnvironmentVariable("OCTAVIA_DATA") is { Length: > 0 } custom)
            return custom;

        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
            if (File.Exists(Path.Combine(dir.FullName, "Octavia.slnx")))
                return Path.Combine(dir.FullName, "data");

        return Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "Octavia");
    }

    /// OCTAVIA_CONFIG points her at a different settings file, so the test harness
    /// can exercise loading and saving without touching the real one.
    public static string ConfigFile =>
        Environment.GetEnvironmentVariable("OCTAVIA_CONFIG") is { Length: > 0 } custom
            ? custom
            : Path.Combine(DataDir, "config.json");
    public static string KeyFile => Path.Combine(DataDir, "apikey.dat");
    /// OCTAVIA_REMOTE_KEY points the remote key at another file, so a check can mint one
    /// and read it back without unpairing every device that trusts the real one.
    public static string RemoteKeyFile =>
        Environment.GetEnvironmentVariable("OCTAVIA_REMOTE_KEY") is { Length: > 0 } custom
            ? custom
            : Path.Combine(DataDir, "remote.key");
    /// OCTAVIA_LOG writes the log somewhere else — the test harness points it at a
    /// temporary file so a check can exercise rotation without destroying the real one.
    public static string LogFile =>
        Environment.GetEnvironmentVariable("OCTAVIA_LOG") is { Length: > 0 } custom
            ? custom
            : Path.Combine(DataDir, "octavia.log");
    public static string FaceRoot => Path.Combine(AppContext.BaseDirectory, "wwwroot");

    /// VRM characters live beside her data, not inside the install, so an avatar
    /// survives an upgrade and can be dropped in without touching Program Files.
    public static string AvatarDir { get; } = Directory.CreateDirectory(
        Path.Combine(DataDir, "avatars")).FullName;
    public static string BrowserData => Path.Combine(DataDir, "WebView2");
}
```
