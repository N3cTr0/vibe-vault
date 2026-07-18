---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ConnectionsInfo.cs
---

# PartnerTool\ConnectionsInfo.cs

```csharp
using System.Diagnostics;

namespace PartnerTool;

public record Connection(string Proto, string Local, string Remote, string State, int Pid, string Process);

/// <summary>
/// Active TCP/UDP connections and listening ports, parsed from <c>netstat -ano</c> with the
/// owning process resolved by PID — quick triage for "what's this machine talking to?".
/// </summary>
public static class ConnectionsInfo
{
    public static async Task<List<Connection>> CollectAsync()
    {
        var text = await ProcessRunner.RunCaptureAsync("netstat.exe", "-ano");
        var names = new Dictionary<int, string>();

        string NameFor(int pid)
        {
            if (names.TryGetValue(pid, out var n)) return n;
            try { using var p = Process.GetProcessById(pid); n = p.ProcessName; }
            catch { n = pid == 0 ? "System Idle" : "—"; }
            names[pid] = n;
            return n;
        }

        var list = new List<Connection>();
        foreach (var raw in text.Split('\n'))
        {
            var parts = raw.Trim().Split(' ', StringSplitOptions.RemoveEmptyEntries);
            if (parts.Length < 4) continue;
            var proto = parts[0];
            if (proto != "TCP" && proto != "UDP") continue;

            // TCP: Proto Local Remote State PID   |   UDP: Proto Local Remote(*:*) PID
            if (proto == "TCP" && parts.Length >= 5 && int.TryParse(parts[4], out var tpid))
                list.Add(new Connection(proto, parts[1], parts[2], parts[3], tpid, NameFor(tpid)));
            else if (proto == "UDP" && parts.Length >= 4 && int.TryParse(parts[^1], out var upid))
                list.Add(new Connection(proto, parts[1], parts[2], "—", upid, NameFor(upid)));
        }

        // Listening + established first, then everything else; cap the list.
        return list
            .OrderByDescending(c => c.State is "LISTENING" or "ESTABLISHED")
            .ThenBy(c => c.Process, StringComparer.OrdinalIgnoreCase)
            .Take(120)
            .ToList();
    }
}
```
