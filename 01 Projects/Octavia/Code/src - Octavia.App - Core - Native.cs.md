---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Core\Native.cs
---

# src\Octavia.App\Core\Native.cs

```csharp
using System.Runtime.InteropServices;

namespace Octavia;

internal static partial class Native
{
    [LibraryImport("user32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static partial bool RegisterHotKey(nint hWnd, int id, uint fsModifiers, uint vk);

    [LibraryImport("user32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static partial bool UnregisterHotKey(nint hWnd, int id);
}
```
