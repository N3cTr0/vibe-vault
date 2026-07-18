---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ActivityLog.cs
---

# PartnerTool\ActivityLog.cs

```csharp
using System.IO;
using System.Reflection;

namespace PartnerTool;

/// <summary>
/// Append-only audit trail of what the tool actually did — every external command it runs and
/// every system-changing action a tech takes — with timestamp and the signed-in user, so an
/// issue can be traced after the fact (e.g. "what did that cleanup touch on the server?").
/// Written to C:\PCI\Logs\PartnerTool_activity.log and persists across runs.
///
/// GOING FORWARD: any new command goes through <see cref="ProcessRunner"/> (auto-logged); any new
/// system-changing action that does NOT shell out (registry/file/WMI/Win32) must call
/// <see cref="Action"/> + <see cref="Result"/> so it lands here too.
/// </summary>
public static class ActivityLog
{
    private static readonly object Gate = new();
    private const  string Dir  = @"C:\PCI\Logs";
    private static readonly string LogFile = Path.Combine(Dir, "PartnerTool_activity.log");
    private static bool _headerWritten;

    /// <summary>A high-level action the tech took (e.g. "Full Repair", "Delete profile jdoe").</summary>
    public static void Action(string category, string message) => Append(category, "» " + message);

    /// <summary>An external command that was run, with its exit code when known.</summary>
    public static void Command(string category, string exe, string args, int? exit = null)
        => Append(category, $"run: {exe} {args}".TrimEnd() + (exit is { } c ? $"  → exit {c}" : ""));

    /// <summary>The outcome of an action (e.g. "Saved", "Freed 8.8 GB", an error message).</summary>
    public static void Result(string category, string message) => Append(category, "   " + message);

    private static void Append(string category, string message)
    {
        try
        {
            lock (Gate)
            {
                Directory.CreateDirectory(Dir);
                if (!_headerWritten)
                {
                    _headerWritten = true;
                    var ver = Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "?";
                    // Logs are written as plain ASCII (UTF-8, no BOM) so they never mojibake when a
                    // tech pastes them into a ticket / RMM tool — see LogText. The in-app view keeps
                    // its nicer glyphs; only the file is flattened.
                    File.AppendAllText(LogFile, LogText.ToAscii(
                        $"{Environment.NewLine}===== Session {DateTime.Now:yyyy-MM-dd HH:mm:ss}  ·  " +
                        $"{Environment.MachineName}\\{Environment.UserName}  ·  PartnerTool {ver} ====={Environment.NewLine}"),
                        LogText.Utf8NoBom);
                }
                File.AppendAllText(LogFile,
                    LogText.ToAscii($"{DateTime.Now:HH:mm:ss} [{category}] {message}{Environment.NewLine}"),
                    LogText.Utf8NoBom);
            }
        }
        catch { /* logging must never throw */ }
    }
}
```
