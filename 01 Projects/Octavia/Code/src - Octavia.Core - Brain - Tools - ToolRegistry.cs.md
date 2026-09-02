---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Tools\ToolRegistry.cs
---

# src\Octavia.Core\Brain\Tools\ToolRegistry.cs

```csharp
using System.Text.Json;
using Octavia.Core;

namespace Octavia.Brain.Tools;

/// One MCP server, as a source of tools.
internal sealed class McpToolProvider(McpClient client) : IToolProvider
{
    private IReadOnlyList<Tool>? _cached;

    public string Name => client.Name;
    public bool IsReady => client.IsRunning;

    public async Task<IReadOnlyList<Tool>> ListAsync(CancellationToken cancel = default)
    {
        // Servers are entitled to change their tool list, but re-asking on every turn
        // costs a round trip in the latency path for something that almost never moves.
        // A restart is what re-reads it, which is also when a server's tools change.
        if (_cached is not null) return _cached;

        try { return _cached = await client.ListToolsAsync(ToolRisk.Act, cancel); }
        catch (Exception ex)
        {
            Log.Warn($"mcp '{Name}': tools/list failed: {ex.Message}");
            return _cached = [];
        }
    }

    public async Task<ToolAnswer> CallAsync(string name, JsonElement arguments, CancellationToken cancel = default)
    {
        // Names are prefixed on the way out so two servers may both offer "get_state";
        // the server itself only knows the bare name.
        var bare = name.StartsWith($"{Name}__", StringComparison.Ordinal)
            ? name[(Name.Length + 2)..]
            : name;

        try { return await client.CallToolAsync(bare, arguments, cancel); }
        catch (Exception ex)
        {
            // Returned, not thrown: a model told what went wrong can say so, where an
            // exception simply ends the turn in silence.
            Log.Warn($"mcp '{Name}': {bare} failed: {ex.Message}");
            return $"That did not work: {ex.Message}";
        }
    }

    public ValueTask DisposeAsync() => client.DisposeAsync();
}

/// Everything she can do, gathered from every provider.
///
/// The registry is what a brain is handed; it never knows how many servers there are or
/// which one answered. That is the whole point of Stage 12 being a seam.
internal sealed class ToolRegistry : IAsyncDisposable
{
    private readonly List<IToolProvider> _providers = [];
    private readonly OctaviaConfig _config;

    public ToolRegistry(OctaviaConfig config) => _config = config;

    public bool Any => _providers.Count > 0;
    public IReadOnlyList<IToolProvider> Providers => _providers;

    /// Starts every configured server. Failures are logged and skipped: one broken
    /// integration must not stop her from talking, or from using the others.
    public async Task StartAsync(CancellationToken cancel = default)
    {
        foreach (var (name, server) in _config.McpServers)
        {
            if (!server.Enabled) { Log.Write($"mcp '{name}': disabled"); continue; }

            if (string.IsNullOrWhiteSpace(server.Command))
            {
                Log.Warn($"mcp '{name}': no command configured");
                continue;
            }

            var client = await McpClient.StartAsync(
                name, server.Command, server.Args ?? [], server.Env, cancel);

            if (client is not null) _providers.Add(new McpToolProvider(client));
        }

        if (_providers.Count > 0)
        {
            var tools = await ListAsync(cancel);
            Log.Write($"tools: {tools.Count} from {_providers.Count} server(s)");
        }
    }

    public async Task<IReadOnlyList<Tool>> ListAsync(CancellationToken cancel = default)
    {
        var all = new List<Tool>();
        foreach (var provider in _providers)
        {
            if (!provider.IsReady) continue;
            all.AddRange(await provider.ListAsync(cancel));
        }
        return all;
    }

    /// Runs a tool, applying the confirmation rule first.
    ///
    /// `confirmed` is set only when the person has said yes *in this conversation*, and
    /// the caller is responsible for having actually asked. Defaulting it to false is
    /// deliberate: a new call site that forgets gets the safe behaviour.
    public async Task<ToolAnswer> CallAsync(
        string name, JsonElement arguments, bool confirmed = false, CancellationToken cancel = default)
    {
        var tool = (await ListAsync(cancel)).FirstOrDefault(t => t.Name == name);
        if (tool is null) return $"There is no tool called {name}.";

        if (tool.Risk == ToolRisk.Confirm && !confirmed)
        {
            Log.Write($"tool '{name}' needs confirmation; not run");

            /* The wording is instructions to the model, and it is worth being exact: it must
               ask *and stop*. Told merely that confirmation is needed, a model will often
               ask and answer itself in the same breath — "shall I unlock it? Yes, unlocking
               now" — which is a confirmation nobody gave. */
            return "Not done: this one needs the person to say yes first. Ask them plainly, " +
                   "in one short question, and say nothing else. Do not assume the answer.";
        }

        var provider = _providers.FirstOrDefault(p => p.Name == tool.Source);
        if (provider is null || !provider.IsReady) return $"{tool.Source} is not available right now.";

        Log.Write($"tool: {name}");
        return await provider.CallAsync(name, arguments, cancel);
    }

    public async ValueTask DisposeAsync()
    {
        foreach (var provider in _providers) await provider.DisposeAsync();
        _providers.Clear();
    }
}
```
