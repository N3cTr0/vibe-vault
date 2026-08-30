---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Core\Tasks.cs
---

# src\Octavia.App\Core\Tasks.cs

```csharp
namespace Octavia.Core;

internal static class Tasks
{
    /// A discarded task swallows its exception until the garbage collector happens to
    /// notice, which is precisely how a subsystem stops working without saying anything.
    /// Every fire-and-forget in the app goes through here so the failure is written down
    /// at the moment it happens.
    public static void Forget(this Task task, string what) =>
        task.ContinueWith(
            faulted => Log.Error($"{what} failed", faulted.Exception!),
            CancellationToken.None,
            TaskContinuationOptions.OnlyOnFaulted,
            TaskScheduler.Default);
}
```
