---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\LoadTimer.cs
---

# PartnerTool\LoadTimer.cs

```csharp
using System.Diagnostics;

namespace PartnerTool;

/// <summary>
/// Times the collectors behind one page load and writes a single activity-log line, slowest first,
/// so a machine in the field names the collector to optimise instead of us guessing from a
/// "that page is slow" report. Wrap each collector with <see cref="Time{T}"/> - a drop-in for
/// <c>Task.Run</c> - and call <see cref="Done"/> when the load finishes.
///
/// Attach it to first-load paths and explicit Refresh clicks only. A repeating timer or a live
/// sampler would fill the log with a line every tick and tell us nothing new.
///
/// The wall time is the number a tech feels; the parts sum to more than it when they ran in
/// parallel, and to roughly it when they ran one after another - which is itself worth seeing.
/// </summary>
public sealed class LoadTimer
{
    private const long SlowMs = 2000;
    private static readonly HashSet<string> Logged = new(StringComparer.OrdinalIgnoreCase);

    private readonly string _page;
    private readonly Stopwatch _wall = Stopwatch.StartNew();
    private readonly List<(string Name, long Ms)> _parts = new();
    private bool _throttle;

    public LoadTimer(string page) => _page = page;

    /// <summary>
    /// For a load that also runs on a timer or on every page re-open: log the first one, then only
    /// loads slow enough to be worth knowing about. Without this, System Info alone would write a
    /// line every 45 seconds and bury the audit trail.
    /// </summary>
    public LoadTimer FirstAndSlowOnly()
    {
        _throttle = true;
        return this;
    }

    /// <summary>Drop-in for <c>await Task.Run(work)</c>. Exceptions propagate unchanged.</summary>
    public async Task<T> Time<T>(string name, Func<T> work)
    {
        var sw = Stopwatch.StartNew();
        try { return await Task.Run(work); }
        finally { Add(name, sw.ElapsedMilliseconds); }
    }

    /// <summary>Drop-in for <c>await Task.Run(work)</c> where the work returns nothing.</summary>
    public async Task Time(string name, Action work)
    {
        var sw = Stopwatch.StartNew();
        try { await Task.Run(work); }
        finally { Add(name, sw.ElapsedMilliseconds); }
    }

    /// <summary>For a collector that is already async - awaited as-is, not re-wrapped in Task.Run.</summary>
    public async Task<T> TimeAsync<T>(string name, Func<Task<T>> work)
    {
        var sw = Stopwatch.StartNew();
        try { return await work(); }
        finally { Add(name, sw.ElapsedMilliseconds); }
    }

    /// <summary>As <see cref="TimeAsync{T}"/>, for async work with no result.</summary>
    public async Task TimeAsync(string name, Func<Task> work)
    {
        var sw = Stopwatch.StartNew();
        try { await work(); }
        finally { Add(name, sw.ElapsedMilliseconds); }
    }

    private void Add(string name, long ms)
    {
        lock (_parts) _parts.Add((name, ms));
    }

    public void Done()
    {
        List<(string Name, long Ms)> parts;
        lock (_parts) parts = new List<(string, long)>(_parts);
        if (parts.Count == 0) return;

        if (_throttle)
        {
            bool first;
            lock (Logged) first = Logged.Add(_page);
            if (!first && _wall.ElapsedMilliseconds < SlowMs) return;
        }

        var detail = string.Join(", ", parts.OrderByDescending(p => p.Ms).Select(p => $"{p.Name} {p.Ms}"));
        ActivityLog.Perf($"{_page} load {_wall.ElapsedMilliseconds} ms  ({detail})");
    }
}
```
