---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Room.cs
---

# src\Octavia.App\Room.cs

```csharp
using Octavia.Brain;
using Octavia.Core;
using Octavia.Face;

namespace Octavia;

/// One space she can be talked to in.
///
/// **One being, N rooms.** The being owns what makes her her — persona, voice, avatar, key,
/// tools, this machine's devices. A room owns what makes a conversation a conversation: its
/// history, its state, its mood, its floor, and whether a camera may open in it.
///
/// Emotion is per room on purpose. It drives the avatar, and a global mood would put an
/// expression on the phone's face that was caused by something said in a different
/// building — which is the whole complaint this exists to answer. Same personality,
/// different mood, the way a person is.
internal sealed class Room : IDisposable
{
    public Room(RoomId id, OctaviaConfig config)
    {
        Id = id;
        Gate = new AttentionGate(config);

        /* Seeded from the config and then owned here. "May she open a camera at all" is a
           question about a place, not about her: the gym phone and the desk should be able
           to answer it differently. Only the host room writes its answer back — the config
           file belongs to this machine. */
        Camera = config.Camera;
        CameraDevice = config.CameraDevice ?? "";
    }

    public RoomId Id { get; }

    /// This room's conversation. Lifted out of `IBrain` so there can be N of these against
    /// one brain, rather than one `ClaudeBrain` per room duplicating the client and the key.
    public Conversation History { get; } = new();

    /// One per room, with no shared statics: the desk being mid-exchange must not make the
    /// phone's gate think it is too. Only a room with always-on listening ever consults it,
    /// which today is the host room alone — push-to-talk bypasses the gate entirely, because
    /// a held button has already answered the question it asks.
    public AttentionGate Gate { get; }

    public AgentState State { get; set; } = AgentState.Idle;
    public Mood Mood { get; set; } = Mood.Neutral;

    public bool Camera { get; set; }
    public string CameraDevice { get; set; }

    public void Dispose() => Gate.Dispose();
}
```
