---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\SecretStore.cs
---

# src\Octavia.Core\Core\SecretStore.cs

```csharp
using System.Security.Cryptography;
using System.Text;

namespace Octavia.Core;

/// The API key never reaches the face. It lives here, DPAPI-sealed to the current user.
internal static class SecretStore
{
    private static readonly byte[] Entropy = Encoding.UTF8.GetBytes("Octavia.ApiKey.v1");

    public static string? ReadApiKey()
    {
        var fromEnv = Environment.GetEnvironmentVariable("ANTHROPIC_API_KEY");
        if (!string.IsNullOrWhiteSpace(fromEnv)) return fromEnv.Trim();

        try
        {
            if (!File.Exists(Paths.KeyFile)) return null;
            var plain = ProtectedData.Unprotect(
                File.ReadAllBytes(Paths.KeyFile), Entropy, DataProtectionScope.CurrentUser);
            return Encoding.UTF8.GetString(plain);
        }
        catch (Exception ex)
        {
            Log.Write($"key read failed: {ex.Message}");
            return null;
        }
    }

    public static void WriteApiKey(string key)
    {
        var sealedKey = ProtectedData.Protect(
            Encoding.UTF8.GetBytes(key.Trim()), Entropy, DataProtectionScope.CurrentUser);
        File.WriteAllBytes(Paths.KeyFile, sealedKey);
    }

    public static void ClearApiKey()
    {
        if (File.Exists(Paths.KeyFile)) File.Delete(Paths.KeyFile);
    }
}
```
