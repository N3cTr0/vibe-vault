---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Brain\Tools\ITool.cs
---

# src\Octavia.App\Brain\Tools\ITool.cs

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;

namespace Octavia.Brain.Tools;

/// How dangerous a tool is, which decides whether she may simply do it.
///
/// This is a property of the *tool*, not of the sentence that reached it. A model that
/// has misheard "turn the lights off" is one token away from having misheard something
/// worse, and the difference between a wrong answer and a dark house is that the second
/// one happened without anybody agreeing to it.
internal enum ToolRisk
{
    /// Reads state and changes nothing. Ask her anything.
    Read,

    /// Changes something reversible and visible — a light, a scene, a volume. She may do
    /// it, and says what she did.
    Act,

    /// Changes something that is awkward or unsafe to undo: locks, alarms, garage doors,
    /// anything that deletes. **Never without an explicit yes in the same conversation.**
    Confirm
}

/// One capability she can use.
internal sealed record Tool(
    string Name,
    string Description,
    JsonObject Schema,
    ToolRisk Risk,
    string Source);

/// Where tools come from. An MCP server is one; a built-in is another.
///
/// The point of the interface is that adding a capability is adding a *provider*, not
/// adding a branch to OctaviaSession — the same bet as IFaceTransport and IBrain, and the
/// reason Stage 12 is one seam rather than N integrations.
internal interface IToolProvider : IAsyncDisposable
{
    string Name { get; }

    /// False when the server is unreachable. She keeps working without it — a house that
    /// cannot be reached is a smaller loss than an assistant that will not talk.
    bool IsReady { get; }

    Task<IReadOnlyList<Tool>> ListAsync(CancellationToken cancel = default);

    /// The result as text for the model. Errors come back as text too, deliberately: a
    /// model that is told "that light does not exist" can say so, where an exception
    /// only ends the turn.
    Task<string> CallAsync(string name, JsonElement arguments, CancellationToken cancel = default);
}
```
