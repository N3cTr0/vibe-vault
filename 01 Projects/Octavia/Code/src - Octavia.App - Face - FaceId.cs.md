---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\FaceId.cs
---

# src\Octavia.App\Face\FaceId.cs

```csharp
using System.Text.Json;

namespace Octavia.Face;

/// Which face. Opaque to the session: it may compare, store and route with one, and can
/// learn nothing from it about how that face connected.
///
/// The transport has always minted one of these per connection — it simply never left
/// `WebSocketFaceServer`. Letting it out is the whole of Stage 14 item 1, and it is what
/// separates "every face" from "that face" without the session learning anything it was
/// deliberately kept from knowing.
internal readonly record struct FaceId(Guid Value)
{
    public static FaceId New() => new(Guid.NewGuid());

    /// Short form, for the log. A full GUID on every line is noise.
    public override string ToString() => Value.ToString("N")[..8];
}

/// One inbound message and the face it came from.
///
/// The body is unchanged — nothing here is visible on the wire, so `PROTOCOL.md` does not
/// move. That is the sign this is a seam being repaired rather than an interface widened.
internal readonly record struct FaceMessage(FaceId From, JsonElement Body);
```
