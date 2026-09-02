---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Tools\ITool.cs
---

# src\Octavia.Core\Brain\Tools\ITool.cs

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

/// What a tool answered with.
///
/// **Text is the whole of it for every tool but one**, which is why this was a bare string
/// until a camera came along. MCP has always allowed a result to carry an image, and the
/// note here used to say that block was "left for whenever that matters" — a snapshot of a
/// front door is when it matters, because the useful answer to *"what is at the gate"* is
/// not a sentence about a JPEG.
///
/// `Image` is base64 exactly as MCP delivers it, and is passed to the model untouched. A
/// brain with no eyes ignores it and uses the text, which is why the text is never optional:
/// **every image-bearing answer also says in words what was captured**, so the local brain
/// gets a usable turn rather than a blank one.
internal sealed record ToolAnswer(string Text, string? Image = null, string? ImageMediaType = null)
{
    public bool HasImage => Image is { Length: > 0 };

    /// The common case, and an implicit conversion so that every existing text-only path
    /// reads exactly as it did.
    public static implicit operator ToolAnswer(string text) => new(text);
}

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

    /// The result for the model — text, and an image when the tool had one to give.
    /// Errors come back as text too, deliberately: a model that is told "that light does
    /// not exist" can say so, where an exception only ends the turn.
    Task<ToolAnswer> CallAsync(string name, JsonElement arguments, CancellationToken cancel = default);
}
```
