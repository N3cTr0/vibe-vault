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

/// Secrets she holds, DPAPI-sealed to the current Windows account.
///
/// The API key was the first and for a long time the only one. A second arrived with the
/// UniFi account — a password for a read-only local admin, because the UniFi *API key*
/// reaches no event history at all and the legacy API that does needs a login. So this is
/// named secrets now rather than one hard-coded file.
///
/// **A password does not go in `config.json`.** The API key in `McpServers.unifi.Env` is
/// already there and already less than ideal; a password is worse, and there was no reason to
/// repeat the mistake to stay consistent with it. Sealed on disk, decrypted only in the
/// server's own memory, and handed to a child process in its environment — see
/// `McpServer.Secrets`, and the note there about arguments being visible in the process list.
internal static class SecretStore
{
    /// **The API key's entropy and filename are unchanged**, and must stay that way: DPAPI
    /// will not open a blob sealed with different entropy, so touching this logs everybody
    /// out of their own key with no way back but pasting it again.
    private const string ApiKeyEntropy = "Octavia.ApiKey.v1";

    public static string? ReadApiKey()
    {
        var fromEnv = Environment.GetEnvironmentVariable("ANTHROPIC_API_KEY");
        if (!string.IsNullOrWhiteSpace(fromEnv)) return fromEnv.Trim();

        return Read(Paths.KeyFile, ApiKeyEntropy);
    }

    public static void WriteApiKey(string key) => Write(Paths.KeyFile, ApiKeyEntropy, key);

    public static void ClearApiKey() => Clear(Paths.KeyFile);

    /// A secret belonging to a tool server, named `<server>:<variable>` — the same pair that
    /// names it in `McpServer.Secrets`, so there is one spelling and not two.
    public static string? ReadFor(string server, string variable) =>
        Read(PathFor(server, variable), EntropyFor(server, variable));

    public static void WriteFor(string server, string variable, string value) =>
        Write(PathFor(server, variable), EntropyFor(server, variable), value);

    public static void ClearFor(string server, string variable) => Clear(PathFor(server, variable));

    public static bool HasFor(string server, string variable) => File.Exists(PathFor(server, variable));

    /// Moves any secret-shaped value out of `config.json` and into the sealed store.
    ///
    /// **The UniFi API key sat in `Env` in plain text for eighteen versions**, and the note
    /// written when `Secrets` was added said so out loud — *"already there and already less
    /// than ideal"* — and then left it. That was the wrong call twice over: the file is
    /// readable by anything running as this user, the settings window drew it on screen in a
    /// text box, and by then the key also authorised **cutting power to a switch port**.
    ///
    /// It is done here rather than asked about because there is no version of this a person
    /// would decline, and doing it needs no input: the value is already known, and sealing it
    /// changes nothing except who can read it.
    ///
    /// Idempotent, and **skipped entirely when running as LocalSystem**. DPAPI seals to an
    /// account, so a service running as the machine would seal a secret the person at the
    /// keyboard could never replace — which is a worse state than the plaintext it fixes.
    ///
    /// Returns the names it sealed, for the log.
    public static IReadOnlyList<string> SealLoose(OctaviaConfig config)
    {
        if (IsLocalSystem())
        {
            foreach (var (server, entry) in config.McpServers)
                foreach (var name in (entry.Env ?? []).Keys.Where(Sensitive.Looks))
                    Log.Warn($"mcp '{server}': {name} is in config.json in plain text, and this " +
                             "process runs as LocalSystem so it cannot be sealed safely. " +
                             "Open her settings as yourself to move it.");

            return [];
        }

        var sealed_ = new List<string>();

        foreach (var (server, entry) in config.McpServers)
        {
            if (entry.Env is not { Count: > 0 }) continue;

            foreach (var name in entry.Env.Keys.Where(Sensitive.Looks).ToList())
            {
                var value = entry.Env[name];
                if (string.IsNullOrWhiteSpace(value)) { entry.Env.Remove(name); continue; }

                try
                {
                    WriteFor(server, name, value);
                }
                catch (Exception ex)
                {
                    // Left exactly where it was. Removing it from `Env` after a failed seal
                    // would lose the value outright, which is far worse than it being readable.
                    Log.Warn($"mcp '{server}': could not seal {name}, leaving it in config.json: {ex.Message}");
                    continue;
                }

                entry.Env.Remove(name);
                entry.Secrets = (entry.Secrets ?? []).Append(name).Distinct().ToArray();
                sealed_.Add($"{server}:{name}");
            }
        }

        if (sealed_.Count > 0) config.Save();
        return sealed_;
    }

    private static bool IsLocalSystem()
    {
        try
        {
            using var identity = System.Security.Principal.WindowsIdentity.GetCurrent();
            return identity.IsSystem;
        }
        catch { return false; }
    }

    /// One file per secret, named after the pair. Lower-cased and stripped of anything that
    /// is not a letter, a digit or a dash, so a server called `UniFi/Network` cannot write
    /// outside the data folder.
    private static string PathFor(string server, string variable) =>
        Path.Combine(Paths.DataDir, $"secret-{Safe(server)}-{Safe(variable)}.dat");

    private static string EntropyFor(string server, string variable) =>
        $"Octavia.Secret.v1:{Safe(server)}:{Safe(variable)}";

    private static string Safe(string name) =>
        new(name.ToLowerInvariant().Select(c => char.IsAsciiLetterOrDigit(c) ? c : '-').ToArray());

    private static string? Read(string file, string entropy)
    {
        try
        {
            if (!File.Exists(file)) return null;

            var plain = ProtectedData.Unprotect(
                File.ReadAllBytes(file), Encoding.UTF8.GetBytes(entropy), DataProtectionScope.CurrentUser);

            return Encoding.UTF8.GetString(plain);
        }
        catch (Exception ex)
        {
            /* Almost always the same cause, and worth naming: **DPAPI seals to a Windows
               account**, so a secret written while logged in as somebody and read back by a
               service running as LocalSystem is unopenable rather than missing. README says
               this about the API key; it is equally true of every secret here. */
            Log.Write($"could not open {Path.GetFileName(file)}: {ex.Message} " +
                      "(a secret is sealed to the account that wrote it)");
            return null;
        }
    }

    private static void Write(string file, string entropy, string value)
    {
        var sealedValue = ProtectedData.Protect(
            Encoding.UTF8.GetBytes(value.Trim()), Encoding.UTF8.GetBytes(entropy), DataProtectionScope.CurrentUser);

        File.WriteAllBytes(file, sealedValue);
    }

    private static void Clear(string file)
    {
        if (File.Exists(file)) File.Delete(file);
    }
}
```
