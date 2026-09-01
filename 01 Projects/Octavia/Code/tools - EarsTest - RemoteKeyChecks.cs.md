---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\RemoteKeyChecks.cs
---

# tools\EarsTest\RemoteKeyChecks.cs

```csharp
// The secret a face from another machine presents. It was broken from v0.14.0 until
// v0.23.1 by a single character: the reader required 24 characters and the writer made
// 23, so every read of Value threw the stored key away and minted a fresh one. Nothing
// ever compared a key against itself, which is why nobody noticed for nine versions.
//
// So the load-bearing assertion here is the dullest one available — mint a key, read it
// back, get the same string. It fails against the old code and it will fail again the
// next time the two halves disagree about what a key looks like.
using Octavia.Core;
using Octavia.Face;

internal static class RemoteKeyChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, bool passed, string detail)
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(passed ? "" : $" — {detail}")}");
            if (!passed) failures++;
        }

        // Her real key is a paired secret: minting one here would silently unpair every
        // device that trusts it, so the whole check runs against a throwaway file.
        var before = Environment.GetEnvironmentVariable("OCTAVIA_REMOTE_KEY");
        var scratch = Path.Combine(Path.GetTempPath(), $"octavia-key-{Guid.NewGuid():N}.key");
        Environment.SetEnvironmentVariable("OCTAVIA_REMOTE_KEY", scratch);

        /* Her log goes somewhere else too, for the same reason FaceProtocolChecks labels
           its server: this check deliberately stores a key too short to be one, which
           makes RemoteKey warn "every paired device will need the new key". Finding that
           in her real log months later, with no run beside it, would send somebody
           looking for an incident that never happened. */
        var beforeLog = Environment.GetEnvironmentVariable("OCTAVIA_LOG");
        var scratchLog = Path.Combine(Path.GetTempPath(), $"octavia-key-{Guid.NewGuid():N}.log");
        Environment.SetEnvironmentVariable("OCTAVIA_LOG", scratchLog);

        try
        {
            File.Delete(scratch);

            // --- the round trip ------------------------------------------
            var minted = RemoteKey.Value;
            var reread = RemoteKey.Value;

            Check("a key survives being read back", minted == reread, $"minted {minted}, re-read {reread}");
            Check("the key reaches the disk", File.Exists(scratch), "no file was written");
            Check("the file holds the key that was returned",
                File.Exists(scratch) && File.ReadAllText(scratch).Trim() == minted,
                File.Exists(scratch) ? File.ReadAllText(scratch).Trim() : "no file");

            // Reading it a third time must not quietly roll it either — the original bug
            // regenerated on *every* access, so two reads were enough to see it and a
            // dozen page assets were enough to make it look random.
            Check("repeated reads are stable", RemoteKey.Value == minted, $"drifted to {RemoteKey.Value}");

            // --- what a key looks like -----------------------------------
            // Asserted from outside rather than against a constant, so a change to the
            // grouping has to be a deliberate change to this line too.
            const string unambiguous = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";
            var bare = minted.Replace("-", "");

            Check("a minted key is grouped for reading", minted.Count(c => c == '-') == 3, minted);
            Check("a minted key is 20 characters", bare.Length == 20, $"{bare.Length}: {minted}");
            Check("a minted key avoids characters that are misread",
                bare.All(unambiguous.Contains), new string(bare.Where(c => !unambiguous.Contains(c)).ToArray()));

            // --- what it accepts -----------------------------------------
            Check("the minted key is accepted", RemoteKey.Matches(minted), minted);
            Check("dashes are not the secret", RemoteKey.Matches(bare), bare);
            Check("a phone keyboard's lower case is accepted", RemoteKey.Matches(minted.ToLowerInvariant()), "rejected");
            Check("spaces from a paste are accepted", RemoteKey.Matches(minted.Replace("-", " ")), "rejected");

            Check("another key is refused", !RemoteKey.Matches("ABCDE-FGHJK-MNPQR-STUVW"), "accepted a wrong key");
            Check("no key is refused", !RemoteKey.Matches(null), "accepted null");
            Check("an empty key is refused", !RemoteKey.Matches("   "), "accepted blank");
            Check("a prefix of the key is refused", !RemoteKey.Matches(bare[..10]), "accepted a prefix");

            // --- rolling it ----------------------------------------------
            var rolled = RemoteKey.Regenerate();
            Check("regenerating changes the key", rolled != minted, "the key did not change");
            Check("a regenerated key round-trips", RemoteKey.Value == rolled, $"wrote {rolled}, read {RemoteKey.Value}");
            Check("regenerating revokes the old key", !RemoteKey.Matches(minted), "the old key still works");
            Check("the new key is accepted", RemoteKey.Matches(rolled), rolled);

            // --- a key somebody typed in by hand -------------------------
            // Rejecting one would silently overwrite it, which unpairs every device
            // without saying so. Long enough is the only bar it has to clear.
            File.WriteAllText(scratch, "MYOWNKEY-THAT-IS-LONG-ENOUGH-TO-BE-ONE");
            Check("a hand-written key of decent length is kept",
                RemoteKey.Value == "MYOWNKEY-THAT-IS-LONG-ENOUGH-TO-BE-ONE", RemoteKey.Value);
            Check("a hand-written key authorises", RemoteKey.Matches("myownkeythatislongenoughtobeone"), "rejected");

            // ...but something too short to be a secret is replaced rather than trusted.
            File.WriteAllText(scratch, "SHORT");
            var replaced = RemoteKey.Value;
            Check("a key too short to be a secret is replaced", replaced != "SHORT", "kept a 5-character key");
            Check("the replacement is itself stable", RemoteKey.Value == replaced, $"drifted to {RemoteKey.Value}");
        }
        finally
        {
            Environment.SetEnvironmentVariable("OCTAVIA_REMOTE_KEY", before);
            Environment.SetEnvironmentVariable("OCTAVIA_LOG", beforeLog);
            try { File.Delete(scratch); } catch { /* a temp file that outlives the run is harmless */ }
            try { File.Delete(scratchLog); } catch { /* likewise */ }
        }

        return failures;
    }
}
```
