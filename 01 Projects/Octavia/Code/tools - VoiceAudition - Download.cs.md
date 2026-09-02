---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Download.cs
---

# tools\VoiceAudition\Download.cs

```csharp
namespace VoiceAudition;

internal static class Download
{
    private static readonly HttpClient Http = new() { Timeout = TimeSpan.FromMinutes(30) };

    /// Downloads beside the target and moves on success - same rule as `PiperStore`, for the
    /// same reason: an interrupted 400 MB fetch must not leave something that looks whole.
    public static async Task ToAsync(string url, string path)
    {
        if (File.Exists(path)) return;

        var partial = path + ".partial";
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        using var response = await Http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
        response.EnsureSuccessStatusCode();

        var total = response.Content.Headers.ContentLength ?? 0;
        var done = 0L;
        var buffer = new byte[1 << 16];
        var lastShown = -1;

        await using (var source = await response.Content.ReadAsStreamAsync())
        await using (var target = File.Create(partial))
        {
            int read;
            while ((read = await source.ReadAsync(buffer)) > 0)
            {
                await target.WriteAsync(buffer.AsMemory(0, read));
                done += read;

                if (total <= 0) continue;
                var percent = (int)(done * 100 / total);
                if (percent == lastShown || percent % 5 != 0) continue;
                lastShown = percent;
                Console.Write($"\r    {percent,3}%  ({done / 1_000_000} of {total / 1_000_000} MB)");
            }
        }

        if (total > 0) Console.WriteLine();
        File.Move(partial, path, overwrite: true);
    }
}
```
