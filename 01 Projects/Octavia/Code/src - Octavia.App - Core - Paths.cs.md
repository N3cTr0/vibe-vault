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
    public static string DataDir { get; } = Directory.CreateDirectory(
        Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "Octavia")).FullName;

    /// OCTAVIA_CONFIG points her at a different settings file, so the test harness
    /// can exercise loading and saving without touching the real one.
    public static string ConfigFile =>
        Environment.GetEnvironmentVariable("OCTAVIA_CONFIG") is { Length: > 0 } custom
            ? custom
            : Path.Combine(DataDir, "config.json");
    public static string KeyFile => Path.Combine(DataDir, "apikey.dat");
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
