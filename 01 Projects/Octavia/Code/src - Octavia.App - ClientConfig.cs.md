---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\ClientConfig.cs
---

# src\Octavia.App\ClientConfig.cs

```csharp
using System.Text.Json;
using Octavia.Core;
using Octavia.Face;

namespace Octavia;

/// Everything a client needs to know, which is very nearly nothing: where she is, and what
/// to say at the door.
///
/// **Deliberately not `OctaviaConfig`.** Her settings describe a microphone, a brain, a
/// voice and a set of tool servers, and a client owns none of those — reading them would
/// invite a client on another machine to believe things about hardware it cannot see. The
/// two files are separate because the two machines might be.
internal sealed class ClientConfig
{
    /// `host:port`, or just `host` for her usual port. Loopback by default, because the
    /// overwhelmingly common case is a server on this machine.
    public string Server { get; set; } = "127.0.0.1:8848";

    /// The remote key. Empty means "read it from the data folder beside me", which is right
    /// whenever the client and the server share a machine and wrong the moment they do not
    /// — so a client on another box states it here and is never surprised.
    public string Key { get; set; } = "";

    /// Which room this client is standing in. Empty is the host room, which is what a
    /// desktop client wants and is also what a face that says nothing already gets.
    public string Room { get; set; } = "";

    /* These two were hers and are now the client's, because that is what they always
       described: a key combination registered with *this* Windows session, and whether
       *this* window starts in the tray. Neither means anything to a server. */

    public string Hotkey { get; set; } = "Ctrl+Alt+O";

    public bool StartMinimised { get; set; }

    public static string FileName => Path.Combine(Paths.DataDir, "client.json");

    /// The same encoder her own settings use, so a hotkey does not come back as
    /// `Ctrl+Alt+O` in a file a person is expected to edit by hand.
    private static readonly JsonSerializerOptions Json = new()
    {
        WriteIndented = true,
        Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping
    };

    /// Missing or unreadable is not an error. A client with no file at all should find a
    /// server on this machine and work, because that is the arrangement almost everybody
    /// has and asking them to write a config file first would be a worse first run than the
    /// one this replaces.
    public static ClientConfig Load()
    {
        try
        {
            if (File.Exists(FileName))
                return JsonSerializer.Deserialize<ClientConfig>(File.ReadAllText(FileName)) ?? new();
        }
        catch (Exception ex)
        {
            Log.Warn($"client.json could not be read ({ex.Message}); using the defaults");
            return new ClientConfig();
        }

        return FirstRun();
    }

    /// The one place a client reads her settings, and it happens once.
    ///
    /// `Hotkey` and `StartMinimised` lived in `config.json` for twenty-five versions, so a
    /// split that simply stopped reading them would silently take away a key combination
    /// somebody chose — the kind of regression that gets reported as "the hotkey broke in
    /// the upgrade" weeks later. Carried across on first run instead, and never read again.
    ///
    /// Her copies are deliberately left in `config.json` rather than deleted: the server
    /// saves that file, and removing the properties would drop them before a client that had
    /// not yet started could ever see them.
    private static ClientConfig FirstRun()
    {
        var client = new ClientConfig();

        try
        {
            if (File.Exists(Paths.ConfigFile))
            {
                using var doc = JsonDocument.Parse(File.ReadAllText(Paths.ConfigFile));

                if (doc.RootElement.TryGetProperty("Hotkey", out var hotkey) &&
                    hotkey.ValueKind == JsonValueKind.String &&
                    hotkey.GetString() is { Length: > 0 } combination)
                    client.Hotkey = combination;

                if (doc.RootElement.TryGetProperty("StartMinimised", out var minimised) &&
                    minimised.ValueKind is JsonValueKind.True or JsonValueKind.False)
                    client.StartMinimised = minimised.GetBoolean();

                Log.Write($"client.json written from her settings; hotkey {client.Hotkey}");
            }
        }
        catch (Exception ex)
        {
            Log.Warn($"could not carry settings over to client.json: {ex.Message}");
        }

        client.Save();
        return client;
    }

    public void Save()
    {
        try
        {
            File.WriteAllText(FileName, JsonSerializer.Serialize(this, Json));
        }
        catch (Exception ex)
        {
            Log.Warn($"client.json could not be written: {ex.Message}");
        }
    }

    /// `host:port` with the port filled in if it was left out.
    ///
    /// Not written to the file. A derived value saved beside the thing it is derived from is
    /// a second copy that will one day disagree with the first, and whoever edits the file
    /// will reasonably expect the one they changed to win.
    [System.Text.Json.Serialization.JsonIgnore]
    public string Authority =>
        Server.Contains(':') ? Server : $"{Server}:8848";

    /// The credential this client presents.
    ///
    /// The **remote key**, never the per-run token: the token is minted fresh on every
    /// server start, so a client holding one would need re-pairing every time she restarted
    /// — which is the same reason a phone uses the key, and `Authorised` has always
    /// accepted it from loopback for exactly this case.
    public string Credential()
    {
        if (Key.Trim().Length > 0) return Key.Trim();

        // Same machine, so the key is simply readable. Reading it also *mints* one on first
        // run, which is what makes a fresh install work with no setup step at all.
        try
        {
            return RemoteKey.Value;
        }
        catch (Exception ex)
        {
            Log.Warn($"no remote key available for the client: {ex.Message}");
            return "";
        }
    }

    /// Where her page is.
    public string PageUrl()
    {
        var url = $"http://{Authority}/index.html?key={Uri.EscapeDataString(Credential())}";
        return Room.Trim().Length > 0 ? $"{url}&room={Uri.EscapeDataString(Room.Trim())}" : url;
    }

    [System.Text.Json.Serialization.JsonIgnore]
    public string Origin => $"http://{Authority}";
}
```
