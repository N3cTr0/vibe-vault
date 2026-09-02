---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Core\Log.cs
---

# src\Octavia.Core\Core\Log.cs

```csharp
namespace Octavia.Core;

internal enum LogLevel { Debug, Info, Warn, Error }

/// One line per meaningful event, kept small enough to read and to email.
///
/// Everything here exists because a failure on someone else's machine has to explain
/// itself in a file. Levels so the interesting lines can be found, rotation so the file
/// stays readable after a month of always-on listening, and a memory of the recent lines
/// so the face can show them without reading the disk.
internal static class Log
{
    /// Past this the file rolls. Small on purpose: the whole point is that it can be
    /// attached to a message. Settable so the harness can prove rotation works without
    /// writing a megabyte to do it.
    public static long MaxBytes { get; set; } = 1024 * 1024;

    /// octavia.1.log through octavia.3.log. Enough to survive a couple of restarts
    /// while someone works out how to reproduce the fault.
    private const int RolledFiles = 3;

    private const int RememberedLines = 300;

    private static readonly object Gate = new();
    private static readonly Queue<string> Remembered = new();

    /// Raise to Debug in config.json when someone needs to reproduce a fault; the
    /// default keeps the file to events a human would recognise.
    public static LogLevel Threshold { get; set; } = LogLevel.Info;

    public static int Errors { get; private set; }
    public static int Warnings { get; private set; }

    public static void Write(string message) => Emit(LogLevel.Info, message);
    public static void Debug(string message) => Emit(LogLevel.Debug, message);
    public static void Warn(string message) => Emit(LogLevel.Warn, message);
    public static void Error(string message) => Emit(LogLevel.Error, message);

    /// Errors carry the whole exception. A message without a stack trace is the thing
    /// you always wish you had when the report arrives from somewhere else.
    public static void Error(string message, Exception ex) => Emit(LogLevel.Error, $"{message}{Environment.NewLine}{ex}");

    /// The most recent lines, oldest first, for the diagnostics panel.
    public static IReadOnlyList<string> Tail(int count)
    {
        lock (Gate) return Remembered.TakeLast(Math.Max(0, count)).ToList();
    }

    /// The log and its rolled predecessors, newest first, for the bundle.
    public static IReadOnlyList<string> Files()
    {
        var files = new List<string>();
        if (File.Exists(Paths.LogFile)) files.Add(Paths.LogFile);
        for (var i = 1; i <= RolledFiles; i++)
        {
            var rolled = RolledPath(i);
            if (File.Exists(rolled)) files.Add(rolled);
        }

        return files;
    }

    public static void SetThreshold(string? name)
    {
        if (Enum.TryParse<LogLevel>(name, ignoreCase: true, out var level)) Threshold = level;
        else if (!string.IsNullOrWhiteSpace(name)) Warn($"unknown LogLevel '{name}'; staying at {Threshold}");
    }

    private static void Emit(LogLevel level, string message)
    {
        if (level < Threshold) return;

        var line = $"{DateTime.Now:MM/dd/yyyy HH:mm:ss}  {Label(level)}  {message}";
        System.Diagnostics.Debug.WriteLine(line);

        lock (Gate)
        {
            if (level == LogLevel.Error) Errors++;
            if (level == LogLevel.Warn) Warnings++;

            Remembered.Enqueue(line);
            while (Remembered.Count > RememberedLines) Remembered.Dequeue();

            Append(line);
        }
    }

    /// One line onto the end of the file, retried briefly when somebody else is mid-write.
    ///
    /// **`lock (Gate)` is a lock inside one process, and there are two of them now.** Since
    /// v0.28.2 the client always starts a server, so a client and a server share this file
    /// as a matter of course — and two processes appending at the same instant is no longer
    /// a rare collision, it is the ordinary case. The old code swallowed the exception, so
    /// the loser simply lost its line: silently, with no trace, in exactly the log somebody
    /// would later read to work out what happened.
    ///
    /// Three quick attempts. The write is short and the contention is microseconds wide, so
    /// this is enough to make the loss theoretical again — and if it still fails, the line
    /// is dropped as before rather than taking her down for it, because a companion that
    /// dies over a log file helps nobody.
    private static void Append(string line)
    {
        for (var attempt = 0; attempt < 3; attempt++)
        {
            try
            {
                Roll();
                File.AppendAllText(Paths.LogFile, line + Environment.NewLine);
                return;
            }
            catch (IOException)
            {
                // The other process has it open. Yield rather than spin: the write it is
                // doing is as short as ours.
                Thread.Sleep(15);
            }
            catch
            {
                // Anything else — a read-only disk, a deleted folder — will not be fixed by
                // trying again. The in-memory tail still holds the line, so the diagnostics
                // panel works even when the disk does not.
                return;
            }
        }
    }

    private static void Roll()
    {
        if (!File.Exists(Paths.LogFile)) return;
        if (new FileInfo(Paths.LogFile).Length < MaxBytes) return;

        var oldest = RolledPath(RolledFiles);
        if (File.Exists(oldest)) File.Delete(oldest);

        for (var i = RolledFiles - 1; i >= 1; i--)
        {
            var from = RolledPath(i);
            if (File.Exists(from)) File.Move(from, RolledPath(i + 1), overwrite: true);
        }

        File.Move(Paths.LogFile, RolledPath(1), overwrite: true);
    }

    private static string RolledPath(int index)
    {
        var directory = Path.GetDirectoryName(Paths.LogFile) ?? Paths.DataDir;
        var name = Path.GetFileNameWithoutExtension(Paths.LogFile);
        return Path.Combine(directory, $"{name}.{index}{Path.GetExtension(Paths.LogFile)}");
    }

    private static string Label(LogLevel level) => level switch
    {
        LogLevel.Debug => "debug",
        LogLevel.Warn => "warn ",
        LogLevel.Error => "ERROR",
        _ => "info "
    };
}
```
