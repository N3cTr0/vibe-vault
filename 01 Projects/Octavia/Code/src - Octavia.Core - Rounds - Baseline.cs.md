---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Rounds\Baseline.cs
---

# src\Octavia.Core\Rounds\Baseline.cs

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using Octavia.Core;

namespace Octavia.Rounds;

/// What normal looks like *here*, learned rather than configured.
///
/// **This exists because of a number.** The first eight-hour sample of the security log held
/// 195 events, every one of them `VERY_HIGH`, at a steady 24 an hour — and 168 of them from a
/// single machine that turned out to be the owner's torrent box. A threshold on severity would
/// have had her announcing something every hour for ever, which is the crying-wolf failure
/// `Finding` was written to avoid, arriving by a route nobody had thought of.
///
/// So severity is not the signal and volume is not the signal. **Change is.**
///
/// > *"I think she should at least learn the system for a week and then start reporting on
/// > them, like if this gets installed at a different location my stuff wouldn't apply there."*
///
/// That is the whole design, and the second half of it is the important half: nothing about
/// any particular network is written down anywhere in this project. A torrent box on one site
/// and a security camera on another are the same thing to this class — a name that turns up
/// often — and neither is named in code or in config.
internal sealed class Baseline
{
    /// A key needs this many observations before it can be judged against, however long the
    /// learning window was. A source seen twice has no normal to depart from.
    private const int EnoughWalks = 12;

    /// How far above its learned ceiling a known source has to go before it is worth saying,
    /// as a multiple and as a floor. **Both, because either alone is wrong:** a multiple alone
    /// makes a source that usually sees 1 event shout at 3, and a floor alone says nothing
    /// about a source that usually sees 400 and now sees 800.
    private const double Multiple = 2.0;
    private const int Floor = 10;

    private readonly string _file;
    private readonly TimeSpan _learnFor;
    private State _state = new();

    public Baseline(string name, TimeSpan learnFor)
    {
        _file = Path.Combine(Paths.DataDir, $"baseline-{name}.json");
        _learnFor = learnFor;
        Load();
    }

    /// When the first observation was made — not when she was installed. A week of learning
    /// means a week of *watching*, so a server that was switched off for three days has three
    /// more days to go.
    public DateTimeOffset? Began => _state.Began;

    public bool Learning(DateTimeOffset now) =>
        _state.Began is not { } began || now - began < _learnFor;

    public TimeSpan Remaining(DateTimeOffset now) =>
        _state.Began is not { } began ? _learnFor : _learnFor - (now - began);

    public int Known => _state.Keys.Count;

    /// Folds one walk's counts in, and says what — if anything — departed from what has been
    /// learned. **During the learning window it always says nothing**, whatever it sees.
    public IReadOnlyList<string> Observe(IReadOnlyDictionary<string, int> counts, DateTimeOffset now)
    {
        _state.Began ??= now;

        var learning = Learning(now);
        var departures = new List<string>();

        if (!learning)
        {
            foreach (var (key, count) in counts)
            {
                if (count <= 0) continue;

                if (!_state.Keys.TryGetValue(key, out var known))
                {
                    // Never seen while learning. This is the one that matters: a machine that
                    // has not set off a single alert in a week and now sets off any.
                    departures.Add($"{key} ({count})");
                    continue;
                }

                if (known.Walks < EnoughWalks) continue;

                var ceiling = Math.Max(known.Most * Multiple, known.Average + Floor);
                if (count > ceiling)
                    departures.Add($"{key} ({count}, usually around {known.Average:0})");
            }
        }

        /* **Only quiet walks are learned from**, once learning is over.

           Folding in a walk that was itself flagged is how a slow escalation teaches her to
           accept it: each hour a little worse than the last, each hour becoming the new
           normal, and nothing ever said. During the learning week everything is folded in,
           because that is what the week is for and there is nothing yet to judge against. */
        if (learning || departures.Count == 0)
        {
            foreach (var (key, count) in counts)
            {
                if (count <= 0) continue;

                var known = _state.Keys.TryGetValue(key, out var existing) ? existing : new Seen();

                known.Walks++;
                known.Total += count;
                known.Most = Math.Max(known.Most, count);
                known.LastAt = now;
                known.FirstAt ??= now;

                _state.Keys[key] = known;
            }

            _state.Walks++;
        }

        Save();
        return departures;
    }

    /// For the health panel, in the words a person would use.
    public string Describe(DateTimeOffset now)
    {
        if (_state.Began is null) return "nothing watched yet";

        if (Learning(now))
        {
            var left = Remaining(now);
            var togo = left.TotalDays >= 1 ? $"{left.TotalDays:0.#} days" : $"{left.TotalHours:0.#} hours";
            return $"learning what is normal — {togo} to go, {Known} source(s) so far";
        }

        return $"watching {Known} known source(s) over {_state.Walks} walk(s)";
    }

    private void Load()
    {
        try
        {
            if (!File.Exists(_file)) return;
            _state = JsonSerializer.Deserialize<State>(File.ReadAllText(_file)) ?? new State();
        }
        catch (Exception ex)
        {
            /* Started again rather than thrown. A corrupt baseline is a week of learning lost,
               which is annoying; refusing to start is worse, and silently judging against half
               a file would be worst of all. */
            Log.Warn($"could not read {Path.GetFileName(_file)}, starting the learning again: {ex.Message}");
            _state = new State();
        }
    }

    private void Save()
    {
        try
        {
            File.WriteAllText(_file, JsonSerializer.Serialize(_state, Json));
        }
        catch (Exception ex)
        {
            Log.Warn($"could not write {Path.GetFileName(_file)}: {ex.Message}");
        }
    }

    private static readonly JsonSerializerOptions Json = new()
    {
        WriteIndented = true,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    /// On disk, and deliberately readable: a person should be able to open it and see what she
    /// thinks is normal, because that is the thing they might disagree with.
    private sealed class State
    {
        public DateTimeOffset? Began { get; set; }
        public int Walks { get; set; }
        public Dictionary<string, Seen> Keys { get; set; } = [];
    }

    private sealed class Seen
    {
        public int Walks { get; set; }
        public long Total { get; set; }
        public int Most { get; set; }
        public DateTimeOffset? FirstAt { get; set; }
        public DateTimeOffset? LastAt { get; set; }

        [JsonIgnore]
        public double Average => Walks == 0 ? 0 : Total / (double)Walks;
    }
}
```
