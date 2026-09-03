---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Tools\McpClient.cs
---

# src\Octavia.Core\Brain\Tools\McpClient.cs

```csharp
using System.Diagnostics;
using System.Text.Json;
using System.Text.Json.Nodes;
using Octavia.Core;

namespace Octavia.Brain.Tools;

/// A Model Context Protocol client, speaking JSON-RPC 2.0 over a child process's stdio.
///
/// **Why MCP rather than an integration each.** Every capability she will ever gain —
/// Home Assistant, the network, a calendar — is the same shape: a list of callable things
/// with typed arguments. MCP already specifies that shape, both Claude and the local
/// models take it, and it keeps each integration in its own process where a hung one
/// cannot take her down with it. A new capability becomes a new server rather than a new
/// branch inside the session.
///
/// Framing is newline-delimited JSON, one message per line, which is what the stdio
/// transport specifies. Nothing here assumes a particular server.
internal sealed class McpClient : IAsyncDisposable
{
    private readonly Process _process;
    private readonly SemaphoreSlim _writing = new(1, 1);
    private readonly Dictionary<int, TaskCompletionSource<JsonElement>> _pending = new();
    private readonly CancellationTokenSource _closing = new();
    private int _nextId;
    private bool _disposed;

    public string Name { get; }
    public bool IsRunning => !_disposed && !_process.HasExited;

    private McpClient(string name, Process process)
    {
        Name = name;
        _process = process;
        _ = Task.Run(ReadLoopAsync);
    }

    /// Starts the server and completes the MCP handshake. Returns null rather than
    /// throwing when it will not start: an integration that is down is a normal state.
    public static async Task<McpClient?> StartAsync(
        string name, string command, IReadOnlyList<string> arguments,
        IReadOnlyDictionary<string, string>? environment, CancellationToken cancel = default)
    {
        try
        {
            var info = new ProcessStartInfo(command)
            {
                RedirectStandardInput = true,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,
                CreateNoWindow = true
            };

            foreach (var argument in arguments) info.ArgumentList.Add(argument);
            if (environment is not null)
                foreach (var (key, value) in environment) info.Environment[key] = value;

            var process = Process.Start(info);
            if (process is null)
            {
                Log.Warn($"mcp '{name}': '{command}' did not start");
                return null;
            }

            // A server's stderr is its diagnostics, not ours; it is logged rather than
            // parsed, and never mixed into the protocol stream on stdout.
            _ = Task.Run(async () =>
            {
                while (await process.StandardError.ReadLineAsync() is { } line)
                    if (line.Length > 0) Log.Write($"mcp '{name}': {line}");
            });

            var client = new McpClient(name, process);

            var initialise = await client.RequestAsync("initialize", new JsonObject
            {
                ["protocolVersion"] = "2025-06-18",
                ["capabilities"] = new JsonObject { ["tools"] = new JsonObject() },
                ["clientInfo"] = new JsonObject { ["name"] = "Octavia", ["version"] = "0.13.0" }
            }, cancel);

            var server = initialise.TryGetProperty("serverInfo", out var serverInfo)
                         && serverInfo.TryGetProperty("name", out var serverName)
                ? serverName.GetString() : "unnamed";

            // The spec requires this after initialize; servers are entitled to withhold
            // everything until they get it, and some do.
            await client.NotifyAsync("notifications/initialized", null, cancel);

            Log.Write($"mcp '{name}': connected to {server}");
            return client;
        }
        catch (Exception ex)
        {
            Log.Warn($"mcp '{name}' would not start: {ex.Message}");
            return null;
        }
    }

    public async Task<IReadOnlyList<Tool>> ListToolsAsync(ToolRisk defaultRisk, CancellationToken cancel = default)
    {
        var result = await RequestAsync("tools/list", new JsonObject(), cancel);
        var tools = new List<Tool>();

        if (!result.TryGetProperty("tools", out var listed) || listed.ValueKind != JsonValueKind.Array)
            return tools;

        foreach (var entry in listed.EnumerateArray())
        {
            var name = entry.TryGetProperty("name", out var n) ? n.GetString() : null;
            if (string.IsNullOrWhiteSpace(name)) continue;

            var description = entry.TryGetProperty("description", out var d) ? d.GetString() ?? "" : "";
            var schema = entry.TryGetProperty("inputSchema", out var s)
                ? JsonNode.Parse(s.GetRawText())?.AsObject() ?? []
                : [];

            /* **The more careful of the two wins, always.**

               The first version of this took a declaration over the guess outright, and the
               comment under it claimed an annotation "can only ever make her more careful" —
               which was true of `destructiveHint` and false of `readOnlyHint`. A server
               claiming `readOnlyHint: true` on a tool the heuristic reads as dangerous would
               have been believed, and a tool that deletes something would have run without
               a question. **A safety claim in a comment that the code does not implement is
               worse than no comment**, because it is what the next person checks instead of
               the code.

               A server may raise the risk of its own tool and may never lower it. Ours are
               annotated because the guess is fragile prose, not because we want it overruled. */
            var guessed = RiskOf(name!, description);
            var declared = DeclaredRisk(entry);
            var risk = declared is { } stated && stated > guessed ? stated : guessed;

            if (declared is null) Log.Debug($"mcp '{Name}': {name} declares no risk; guessed {risk}");
            else if (declared < guessed)
                Log.Warn($"mcp '{Name}': {name} claims {declared} but reads as {guessed}; keeping {guessed}");

            tools.Add(new Tool($"{Name}__{name}", description, schema, risk, Name));
        }

        return tools;
    }

    /// The risk a server states about its own tool, via MCP's standard `annotations`.
    ///
    /// **A server knows what its tool does; `RiskOf` is guessing.** The guess reads the
    /// description for dangerous words, which means a tool is dangerous for as long as
    /// nobody rewrites its prose — `set_port_power` classified as `Confirm` only because a
    /// sentence pointing at the other tool happened to contain "reboot". That is not a
    /// property anything should depend on when the cost of a wrong answer is cutting power
    /// to a camera without asking.
    ///
    /// Only `destructiveHint: true` and `readOnlyHint: true` are honoured, and only those.
    /// A *missing* annotation falls through to the guess rather than being read as "safe",
    /// because most servers in the world do not set one, and `destructiveHint: false` is
    /// ignored outright.
    ///
    /// **This function only reports what was claimed.** Whether the claim is taken is the
    /// caller's decision, and the caller keeps whichever of the two is more careful — so a
    /// `readOnlyHint` on something the wording reads as dangerous is logged and discarded.
    private static ToolRisk? DeclaredRisk(JsonElement entry)
    {
        if (!entry.TryGetProperty("annotations", out var notes) || notes.ValueKind != JsonValueKind.Object)
            return null;

        if (notes.TryGetProperty("destructiveHint", out var destructive) &&
            destructive.ValueKind == JsonValueKind.True)
            return ToolRisk.Confirm;

        if (notes.TryGetProperty("readOnlyHint", out var readOnly) &&
            readOnly.ValueKind == JsonValueKind.True)
            return ToolRisk.Read;

        return null;
    }

    /// Guessing risk from a name is crude, and it is deliberately biased towards asking.
    ///
    /// MCP has no risk annotation, so the alternative to a heuristic is treating every
    /// tool as safe — which is how an assistant unlocks a door because a sentence on the
    /// television resembled a request. A read wrongly classed as dangerous costs one
    /// needless question; an unlock wrongly classed as safe costs rather more.
    internal static ToolRisk RiskOf(string name, string description)
    {
        var text = $"{name} {description}".ToLowerInvariant();

        if (text.ContainsAny("unlock", "lock", "disarm", "arm", "alarm", "garage", "door",
                             "delete", "remove", "reset", "reboot", "restart", "shutdown",
                             "purge", "wipe", "factory", "payment", "buy", "order"))
            return ToolRisk.Confirm;

        if (text.ContainsAny("get_", "list_", "read", "status", "state", "query", "search",
                             "history", "sensor", "who", "what"))
            return ToolRisk.Read;

        return ToolRisk.Act;
    }

    public async Task<ToolAnswer> CallToolAsync(string bareName, JsonElement arguments, CancellationToken cancel = default)
    {
        var parameters = new JsonObject
        {
            ["name"] = bareName,
            ["arguments"] = arguments.ValueKind == JsonValueKind.Undefined
                ? new JsonObject()
                : JsonNode.Parse(arguments.GetRawText())
        };

        var result = await RequestAsync("tools/call", parameters, cancel);

        /* MCP returns content blocks. The text ones are what any model can use; an image is
           taken too now, because a camera answers a question that no sentence about a JPEG
           ever could. **Only the first image is kept** — a tool that returned twelve frames
           would otherwise put twelve full-size images into one turn, which is a bill and a
           context window rather than an answer. */
        if (result.TryGetProperty("content", out var content) && content.ValueKind == JsonValueKind.Array)
        {
            var blocks = content.EnumerateArray().ToList();

            var text = blocks
                .Where(b => b.TryGetProperty("type", out var t) && t.GetString() == "text")
                .Select(b => b.TryGetProperty("text", out var v) ? v.GetString() : null)
                .Where(v => !string.IsNullOrWhiteSpace(v));

            var joined = string.Join("\n", text);

            var picture = blocks.FirstOrDefault(b =>
                b.TryGetProperty("type", out var t) && t.GetString() == "image" &&
                b.TryGetProperty("data", out _));

            if (picture.ValueKind == JsonValueKind.Object)
            {
                var data = picture.GetProperty("data").GetString();
                var kind = picture.TryGetProperty("mimeType", out var m) ? m.GetString() : null;

                if (data is { Length: > 0 })
                {
                    Log.Write($"tool '{bareName}' returned an image ({data.Length / 1365} KB)");

                    // The words matter as much as the picture: a brain with no eyes gets only
                    // this, and "an image" is a worse answer than "the Back Garden camera".
                    return new ToolAnswer(
                        joined.Length > 0 ? joined : $"{bareName} returned an image.",
                        data,
                        kind is { Length: > 0 } ? kind : "image/jpeg");
                }
            }

            if (joined.Length > 0) return joined;
        }

        return result.GetRawText();
    }

    // ---- transport ----------------------------------------------

    private async Task<JsonElement> RequestAsync(string method, JsonNode? parameters, CancellationToken cancel)
    {
        var id = Interlocked.Increment(ref _nextId);
        var waiting = new TaskCompletionSource<JsonElement>(TaskCreationOptions.RunContinuationsAsynchronously);
        lock (_pending) _pending[id] = waiting;

        await SendAsync(new JsonObject
        {
            ["jsonrpc"] = "2.0",
            ["id"] = id,
            ["method"] = method,
            ["params"] = parameters
        }, cancel);

        // A server that never answers must not hang a turn for ever.
        using var timeout = CancellationTokenSource.CreateLinkedTokenSource(cancel, _closing.Token);
        timeout.CancelAfter(TimeSpan.FromSeconds(30));

        await using (timeout.Token.Register(() => waiting.TrySetCanceled()))
            return await waiting.Task;
    }

    private Task NotifyAsync(string method, JsonNode? parameters, CancellationToken cancel) =>
        SendAsync(new JsonObject { ["jsonrpc"] = "2.0", ["method"] = method, ["params"] = parameters }, cancel);

    private async Task SendAsync(JsonObject message, CancellationToken cancel)
    {
        await _writing.WaitAsync(cancel);
        try
        {
            await _process.StandardInput.WriteLineAsync(message.ToJsonString());
            await _process.StandardInput.FlushAsync(cancel);
        }
        finally
        {
            _writing.Release();
        }
    }

    private async Task ReadLoopAsync()
    {
        try
        {
            while (await _process.StandardOutput.ReadLineAsync() is { } line)
            {
                if (line.Length == 0) continue;

                JsonElement message;
                try { message = JsonDocument.Parse(line).RootElement.Clone(); }
                catch { Log.Write($"mcp '{Name}': unparseable line"); continue; }

                if (!message.TryGetProperty("id", out var idNode) || !idNode.TryGetInt32(out var id))
                    continue;   // a notification from the server; nothing waits on it

                TaskCompletionSource<JsonElement>? waiting;
                lock (_pending) { _pending.Remove(id, out waiting); }
                if (waiting is null) continue;

                if (message.TryGetProperty("error", out var error))
                {
                    var text = error.TryGetProperty("message", out var m) ? m.GetString() : "unknown error";
                    waiting.TrySetException(new InvalidOperationException($"{Name}: {text}"));
                }
                else
                {
                    waiting.TrySetResult(message.TryGetProperty("result", out var result)
                        ? result.Clone()
                        : default);
                }
            }
        }
        catch (Exception ex) when (!_disposed)
        {
            Log.Warn($"mcp '{Name}' read loop ended: {ex.Message}");
        }

        // Whatever is still waiting will never be answered now.
        lock (_pending)
        {
            foreach (var waiting in _pending.Values)
                waiting.TrySetException(new InvalidOperationException($"{Name} disconnected"));
            _pending.Clear();
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;
        await _closing.CancelAsync();

        try
        {
            if (!_process.HasExited)
            {
                _process.StandardInput.Close();
                if (!_process.WaitForExit(2000)) _process.Kill(entireProcessTree: true);
            }
        }
        catch (Exception ex)
        {
            Log.Write($"mcp '{Name}' shutdown: {ex.Message}");
        }

        _process.Dispose();
        _writing.Dispose();
        _closing.Dispose();
    }
}

internal static class TextMatch
{
    /// True when any needle appears **at the start of a word**.
    ///
    /// This was a plain substring match, and a tool for reading the security log — described,
    /// accurately, as reporting intrusion attempts the gateway *blocked* — was classified as
    /// `Confirm`, because **"blocked" contains "lock"**. The consequence was not a needless
    /// question: a `Confirm` tool does not run without a person saying yes, and nothing was
    /// going to ask on an hourly round, so the feature would simply never have worked.
    ///
    /// Word-*start* rather than whole-word, deliberately. "lock" must still catch "locks" and
    /// "locking", so an ending is allowed and a beginning is not: `clock` and `blocked` are
    /// other words, `locking` is the same one. It also retired the `"arm "` hack, whose
    /// trailing space existed to stop it matching "alarm" and "warm".
    public static bool ContainsAny(this string text, params string[] needles) =>
        needles.Any(needle => StartsAWord(text, needle));

    private static bool StartsAWord(string text, string needle)
    {
        for (var at = text.IndexOf(needle, StringComparison.Ordinal); at >= 0;
             at = text.IndexOf(needle, at + 1, StringComparison.Ordinal))
        {
            // The character before it must not be part of a word. `get_` and the like are
            // needles with punctuation in them, and start a word the same way.
            if (at == 0 || !char.IsLetterOrDigit(text[at - 1])) return true;
        }

        return false;
    }
}
```
