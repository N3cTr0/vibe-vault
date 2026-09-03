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
    /// Past this a single day's file rolls to `octavia-2026-09-03.1.log`. Small on purpose:
    /// the whole point is that a log can be attached to a message. Settable so the harness can
    /// prove rotation works without writing a megabyte to do it.
    ///
    /// **Secondary to the date since v0.45.0**, and kept rather than replaced by it: a day
    /// is a good unit for finding things and no unit at all for bounding size, and the day
    /// something goes wrong at three in the morning is the day it writes ten gigabytes.
    public static long MaxBytes { get; set; } = 1024 * 1024;

    /// Rolled files *within one day*. Enough to survive a couple of restarts while someone
    /// works out how to reproduce the fault.
    private const int RolledFiles = 3;

    /// How many days of logs are kept. Older ones are deleted on the first write of each day.
    /// Zero or less keeps everything, which is a real answer for somebody chasing a fault
    /// across a fortnight and a bad default for everybody else.
    public static int KeepDays { get; set; } = 14;

    /// The day the last purge ran, so it happens once at midnight rather than on every line.
    ///
    /// Settable only so a check can pretend the day has turned over; there is no other reason
    /// to write to it, and a running server never does.
    internal static DateOnly Purged { get; set; } = DateOnly.MinValue;

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

    /// **Today's file.** `Paths.LogFile` is the base name and never written to directly since
    /// v0.45.0; the day is spliced in, so midnight rotates the log by simply being a different
    /// answer to this question. Nothing schedules anything, and a server that was asleep at
    /// midnight rotates on its first line of the new day exactly as one that was awake.
    public static string Today => DatedPath(DateTime.Now);

    /// Every log file that survives, newest first, for the bundle. Days rather than a fixed
    /// count of rolled files: *"the last fortnight"* is a thing a person can reason about and
    /// "octavia.3.log" is not.
    public static IReadOnlyList<string> Files()
    {
        try
        {
            var directory = Path.GetDirectoryName(Paths.LogFile) ?? Paths.DataDir;
            var stem = Path.GetFileNameWithoutExtension(Paths.LogFile);

            return Directory.EnumerateFiles(directory, $"{stem}*{Path.GetExtension(Paths.LogFile)}")
                            .OrderByDescending(File.GetLastWriteTimeUtc)
                            .ToList();
        }
        catch (Exception ex)
        {
            System.Diagnostics.Debug.WriteLine($"could not list log files: {ex.Message}");
            return File.Exists(Today) ? [Today] : [];
        }
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
                var today = Today;

                Purge();
                Roll(today);
                File.AppendAllText(today, line + Environment.NewLine);
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

    /// Deletes logs older than `KeepDays`, once per day.
    ///
    /// **By the file's own timestamp rather than by reading a date out of its name**, which
    /// costs nothing and quietly does the right thing with the `octavia.log` and
    /// `octavia.1.log` left behind by every version before this one. A purge that only
    /// understood the new naming would have left those on disk for ever, which is exactly the
    /// waste this was asked for to prevent.
    private static void Purge()
    {
        var today = DateOnly.FromDateTime(DateTime.Now);
        if (Purged == today || KeepDays <= 0) return;

        // Stamped before the work, not after: a purge that throws must not run again on
        // every single line for the rest of the day.
        Purged = today;

        try
        {
            var directory = Path.GetDirectoryName(Paths.LogFile) ?? Paths.DataDir;
            var stem = Path.GetFileNameWithoutExtension(Paths.LogFile);
            var extension = Path.GetExtension(Paths.LogFile);
            var cutoff = DateTime.Now.AddDays(-KeepDays);
            var keep = Today;

            foreach (var file in Directory.EnumerateFiles(directory, $"{stem}*{extension}"))
            {
                // Never today's, whatever its timestamp says. A clock that jumped backwards
                // should cost somebody a confusing filename, not the log they are writing.
                if (string.Equals(file, keep, StringComparison.OrdinalIgnoreCase)) continue;
                if (File.GetLastWriteTime(file) >= cutoff) continue;

                File.Delete(file);
                System.Diagnostics.Debug.WriteLine($"purged old log {Path.GetFileName(file)}");
            }
        }
        catch (Exception ex)
        {
            // Not through `Warn`: this runs from inside `Append`, and logging about the log
            // from inside the log is how a stack overflow gets written.
            System.Diagnostics.Debug.WriteLine($"could not purge old logs: {ex.Message}");
        }
    }

    private static void Roll(string today)
    {
        if (!File.Exists(today)) return;
        if (new FileInfo(today).Length < MaxBytes) return;

        var oldest = RolledPath(today, RolledFiles);
        if (File.Exists(oldest)) File.Delete(oldest);

        for (var i = RolledFiles - 1; i >= 1; i--)
        {
            var from = RolledPath(today, i);
            if (File.Exists(from)) File.Move(from, RolledPath(today, i + 1), overwrite: true);
        }

        File.Move(today, RolledPath(today, 1), overwrite: true);
    }

    /// `octavia-2026-09-03.log`, from the base name in `Paths.LogFile`.
    ///
    /// Derived rather than hard-coded so `OCTAVIA_LOG` still redirects the whole scheme, which
    /// is what lets a check exercise a fortnight of rotation without touching the real one.
    private static string DatedPath(DateTime day)
    {
        var directory = Path.GetDirectoryName(Paths.LogFile) ?? Paths.DataDir;
        var stem = Path.GetFileNameWithoutExtension(Paths.LogFile);
        return Path.Combine(directory, $"{stem}-{day:yyyy-MM-dd}{Path.GetExtension(Paths.LogFile)}");
    }

    private static string RolledPath(string dated, int index)
    {
        var directory = Path.GetDirectoryName(dated) ?? Paths.DataDir;
        var name = Path.GetFileNameWithoutExtension(dated);
        return Path.Combine(directory, $"{name}.{index}{Path.GetExtension(dated)}");
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
