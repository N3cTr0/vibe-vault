---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\McpServer.cs
---

# src\Octavia.Core\Core\McpServer.cs

```csharp
namespace Octavia.Core;

/// One MCP server she may start and call.
///
/// Kept deliberately close to the shape every other MCP client uses — command, args,
/// env — so a server someone else wrote and documented can be pasted in without
/// translation, and so the servers that already exist for Home Assistant and for network
/// gear are usable as they are.
internal sealed class McpServer
{
    /// The executable. `npx`, `uv`, `python`, or a path to something built here.
    public string Command { get; set; } = "";

    public string[]? Args { get; set; }

    /// Environment for the child process. **This is where tokens go.** A secret in an
    /// argument is visible in the process list to every account on the machine; one in
    /// the environment is not, and neither ends up in her log.
    public Dictionary<string, string>? Env { get; set; }

    /// Off without deleting the entry, so a misbehaving integration can be silenced
    /// without losing how it was configured.
    public bool Enabled { get; set; } = true;
}
```
