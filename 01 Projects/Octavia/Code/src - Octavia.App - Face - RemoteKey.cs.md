---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\RemoteKey.cs
---

# src\Octavia.App\Face\RemoteKey.cs

```csharp
using System.Security.Cryptography;
using Octavia.Core;

namespace Octavia.Face;

/// The secret a face from another machine has to present.
///
/// **Why this is not the per-run token.** That token is regenerated every start, which is
/// exactly right for a page the host itself loads a second later and useless for a phone
/// in a pocket: it would need re-pairing every time she restarted. This one is written
/// once and survives.
///
/// It is stored in her data folder in the clear, deliberately — it has to be *readable*
/// so it can be shown in Settings and typed into a phone, which rules out DPAPI. That is
/// an honest trade and it is why the key alone is not the security boundary: remote
/// access is off by default, and when it is on the socket should be reachable only over
/// Tailscale or Wireguard, never a forwarded port. See ROADMAP.md stage 13.
internal static class RemoteKey
{
    private static string Path => System.IO.Path.Combine(Paths.DataDir, "remote.key");

    /// The current key, creating one on first use.
    public static string Value
    {
        get
        {
            try
            {
                if (File.Exists(Path))
                {
                    var existing = File.ReadAllText(Path).Trim();
                    if (existing.Length >= 24) return existing;
                }
            }
            catch (Exception ex)
            {
                Log.Warn($"could not read the remote key: {ex.Message}");
            }

            return Regenerate();
        }
    }

    /// A new key, which immediately invalidates every device paired with the old one.
    /// That is the revocation story: there is no per-device list to prune, so losing a
    /// phone means rolling this and re-entering it on the ones you still have.
    public static string Regenerate()
    {
        // Base32-ish over an unambiguous alphabet: this gets read off a screen and typed
        // on a phone, so 0/O and 1/I/l must not both exist in it.
        const string alphabet = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";
        var bytes = RandomNumberGenerator.GetBytes(20);
        var key = string.Concat(bytes.Select(b => alphabet[b % alphabet.Length]));

        // Grouped, because a twenty-character string with no shape in it is read wrongly.
        var grouped = string.Join("-", Enumerable.Range(0, 4).Select(i => key.Substring(i * 5, 5)));

        try
        {
            File.WriteAllText(Path, grouped);
            Log.Write("remote key written");
        }
        catch (Exception ex)
        {
            Log.Error("could not write the remote key", ex);
        }

        return grouped;
    }

    /// Fixed-time, and normalised so the dashes and the case a phone keyboard produces
    /// do not decide whether someone gets in.
    public static bool Matches(string? offered)
    {
        if (string.IsNullOrWhiteSpace(offered)) return false;

        static byte[] Normalise(string s) =>
            System.Text.Encoding.UTF8.GetBytes(s.Replace("-", "").Replace(" ", "").ToUpperInvariant());

        return CryptographicOperations.FixedTimeEquals(Normalise(offered), Normalise(Value));
    }
}
```
