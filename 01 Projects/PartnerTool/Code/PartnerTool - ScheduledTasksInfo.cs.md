---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ScheduledTasksInfo.cs
---

# PartnerTool\ScheduledTasksInfo.cs

```csharp
namespace PartnerTool;

public record ScheduledTaskItem(string Name, string Status, string NextRun, string LastRun, string LastResult);

/// <summary>
/// Scheduled tasks, parsed from <c>schtasks /query /fo CSV /v</c> (built-in). Useful for
/// spotting persistence/malware and "what's supposed to run on this box".
/// </summary>
public static class ScheduledTasksInfo
{
    public static async Task<List<ScheduledTaskItem>> CollectAsync()
    {
        var list = new List<ScheduledTaskItem>();
        try
        {
            var text = await ProcessRunner.RunCaptureAsync("schtasks.exe", "/query /fo CSV /v", 30000);
            var lines = text.Split('\n');
            int taskCol = -1, statusCol = -1, nextCol = -1, lastCol = -1, resultCol = -1;
            var seen = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

            foreach (var raw in lines)
            {
                var line = raw.TrimEnd('\r');
                if (line.Length == 0) continue;
                var cols = SplitCsv(line);
                if (cols.Count == 0) continue;

                // Header row repeats throughout the output; use it to find columns.
                if (cols[0].Equals("TaskName", StringComparison.OrdinalIgnoreCase))
                {
                    taskCol   = cols.FindIndex(c => c.Equals("TaskName", StringComparison.OrdinalIgnoreCase));
                    statusCol = cols.FindIndex(c => c.Equals("Status", StringComparison.OrdinalIgnoreCase));
                    nextCol   = cols.FindIndex(c => c.Equals("Next Run Time", StringComparison.OrdinalIgnoreCase));
                    lastCol   = cols.FindIndex(c => c.Equals("Last Run Time", StringComparison.OrdinalIgnoreCase));
                    resultCol = cols.FindIndex(c => c.Equals("Last Result", StringComparison.OrdinalIgnoreCase));
                    continue;
                }
                if (taskCol < 0 || taskCol >= cols.Count) continue;

                var name = cols[taskCol];
                if (string.IsNullOrWhiteSpace(name) || !seen.Add(name)) continue;
                list.Add(new ScheduledTaskItem(
                    name,
                    Get(cols, statusCol), Get(cols, nextCol), Get(cols, lastCol), Get(cols, resultCol)));
            }
        }
        catch { }
        return list.OrderBy(t => t.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    private static string Get(List<string> cols, int i) => i >= 0 && i < cols.Count ? cols[i] : "";

    private static List<string> SplitCsv(string line)
    {
        var result = new List<string>();
        var cur = new System.Text.StringBuilder();
        bool inQuotes = false;
        for (int i = 0; i < line.Length; i++)
        {
            char c = line[i];
            if (c == '"')
            {
                if (inQuotes && i + 1 < line.Length && line[i + 1] == '"') { cur.Append('"'); i++; }
                else inQuotes = !inQuotes;
            }
            else if (c == ',' && !inQuotes) { result.Add(cur.ToString()); cur.Clear(); }
            else cur.Append(c);
        }
        result.Add(cur.ToString());
        return result;
    }
}
```
