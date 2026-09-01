---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Face\RoomId.cs
---

# src\Octavia.Core\Face\RoomId.cs

```csharp
namespace Octavia.Face;

/// Which space a conversation is happening in. Faces in the same room see the same
/// conversation; faces in different rooms share nothing but her.
///
/// **A room is not a face.** The Android client opens two connections — a native one that
/// owns the microphone and a WebView panel that draws her page — and both are the phone's
/// room. If room and face were the same thing she would be talking to herself in the next
/// tab.
internal readonly record struct RoomId(string Value)
{
    /// The machine she runs on. Faces that name no room land here, so the built-in page
    /// and every existing renderer keep today's behaviour exactly.
    public static readonly RoomId Host = new("host");

    /// Trimmed and folded, because a room is typed by a person into a query string and
    /// `?room=Phone` and `?room=phone ` are the same intent. Anything empty is the host.
    public static RoomId Named(string? value) =>
        string.IsNullOrWhiteSpace(value) ? Host : new(value.Trim().ToLowerInvariant());

    public override string ToString() => Value;
}
```
