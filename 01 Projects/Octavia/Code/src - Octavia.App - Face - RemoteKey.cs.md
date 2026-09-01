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
    // Base32-ish over an unambiguous alphabet: this gets read off a screen and typed on
    // a phone, so 0/O and 1/I/l must not both exist in it.
    private const string Alphabet = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";

    /* The shape lives here once, and everything else is *derived* from it. Until v0.23.1
       the reader wanted 24 characters and the writer made 23 — four groups of five and
       three dashes — so no generated key could ever satisfy the guard. Every read of
       Value therefore threw the stored key away and minted a new one, and no remote face
       could ever authenticate: it offered a key, Matches read Value, Value replaced it,
       and the comparison was against a secret one microsecond old.

       The bug was not the number. It was that a human worked the number out by hand from
       a format defined somewhere else, so the two halves could drift apart silently. A
       constant that is computed cannot drift, and the round-trip check in EarsTest fails
       loudly if this ever stops being true. */
    private const int Groups = 4;
    private const int GroupSize = 5;
    private const int Chars = Groups * GroupSize;

    private static string Path => Paths.RemoteKeyFile;

    /// Dashes, spaces and the case a phone keyboard produces are presentation, not
    /// secret. Both the length guard and the comparison go through here, so they cannot
    /// disagree about what part of the string actually matters.
    private static string Normalise(string s) =>
        s.Replace("-", "").Replace(" ", "").ToUpperInvariant();

    /// The current key, creating one on first use.
    ///
    /// Deliberately re-read from disk rather than cached: a key edited by hand takes
    /// effect at once, and the round-trip check is then testing the file rather than a
    /// field that would agree with itself no matter what was written.
    public static string Value
    {
        get
        {
            try
            {
                if (File.Exists(Path))
                {
                    var existing = File.ReadAllText(Path).Trim();

                    /* Length *after* normalising, and only a floor. A key somebody typed
                       in by hand is a legitimate thing to find here — the file is plain
                       text on purpose — and rejecting one would overwrite it silently,
                       which unpairs every device without ever saying so. Too short to be
                       a secret is the only thing worth refusing. */
                    if (Normalise(existing).Length >= Chars) return existing;

                    Log.Warn("the stored remote key is too short to be one; minting a new one — " +
                             "every paired device will need the new key");
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
        var bytes = RandomNumberGenerator.GetBytes(Chars);
        var key = string.Concat(bytes.Select(b => Alphabet[b % Alphabet.Length]));

        // Grouped, because a twenty-character string with no shape in it is read wrongly.
        var grouped = string.Join("-",
            Enumerable.Range(0, Groups).Select(i => key.Substring(i * GroupSize, GroupSize)));

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

        static byte[] Bytes(string s) => System.Text.Encoding.UTF8.GetBytes(Normalise(s));

        return CryptographicOperations.FixedTimeEquals(Bytes(offered), Bytes(Value));
    }
}
```
