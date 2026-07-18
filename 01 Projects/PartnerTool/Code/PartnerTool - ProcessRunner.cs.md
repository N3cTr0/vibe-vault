---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ProcessRunner.cs
---

# PartnerTool\ProcessRunner.cs

```csharp
using System.Diagnostics;
using System.IO;
using System.Text;
using System.Text.RegularExpressions;

namespace PartnerTool;

/// <summary>
/// Runs a console process with redirected output, cleans the noise (ANSI codes,
/// carriage-return overwrites, null bytes from UTF-16 tools like sfc), and
/// surfaces percentage progress from tools that draw a progress bar (DISM).
/// Output/error callbacks fire on a thread-pool thread — marshal to the UI yourself.
/// </summary>
public static class ProcessRunner
{
    private static readonly Regex Ansi    = new(@"\x1B\[[0-9;]*[A-Za-z]|\x1B.", RegexOptions.Compiled);
    private static readonly Regex Percent = new(@"(\d{1,3})(?:[.,]\d+)?\s*%", RegexOptions.Compiled);

    /// <summary>
    /// How long to keep reading output after the process itself has exited. Whatever the tool wrote
    /// is already in the pipe by then, so this only needs to cover the reader flushing it — we are
    /// not waiting for a lingering grandchild to close the handle (see RunAsync).
    /// </summary>
    private const int DrainMs = 1500;

    /// <summary>
    /// Resolve a bare system-tool name (e.g. "powershell.exe", "powercfg") to its absolute path
    /// under System32. We run elevated, and CreateProcess searches the app folder and the current
    /// directory before System32 — so a bare name could be hijacked by a planted binary if the app
    /// is run from a writable folder. Callers that pass a full path get it back unchanged.
    /// </summary>
    public static string ResolveSystemExe(string exe)
    {
        if (string.IsNullOrEmpty(exe) || exe.Contains('\\') || exe.Contains('/')) return exe;

        var sys = Environment.GetFolderPath(Environment.SpecialFolder.System);
        if (exe.Equals("powershell.exe", StringComparison.OrdinalIgnoreCase) ||
            exe.Equals("powershell",     StringComparison.OrdinalIgnoreCase))
        {
            var ps = Path.Combine(sys, @"WindowsPowerShell\v1.0\powershell.exe");
            return File.Exists(ps) ? ps : exe;
        }
        var name = exe.EndsWith(".exe", StringComparison.OrdinalIgnoreCase) ? exe : exe + ".exe";
        var full = Path.Combine(sys, name);
        if (File.Exists(full)) return full;

        // Not every system tool sits directly in System32 — winmgmt.exe lives in System32\wbem.
        // Without this it fell through to the bare name, which is the very hijack we're guarding
        // against (an elevated CreateProcess searches the app folder and CWD before PATH).
        var wbem = Path.Combine(sys, "wbem", name);
        if (File.Exists(wbem)) return wbem;

        return exe;   // not a shipped system tool (e.g. winget) → leave as-is
    }

    /// <summary>
    /// Pin ANY system tool file (exe, msc, cpl…) to its absolute path under System32, or the
    /// Windows directory as fallback (regedit.exe lives there). Needed for ShellExecute launches:
    /// its search order includes the working directory, so an elevated launch of a bare name could
    /// run a file planted next to wherever the exe was started from (e.g. Downloads). Full paths
    /// and URI schemes pass through unchanged.
    /// </summary>
    public static string ResolveSystemTool(string file)
    {
        if (string.IsNullOrEmpty(file) || file.Contains('\\') || file.Contains('/') || file.Contains(':'))
            return file;   // already a path, or a URI like ms-settings:
        var sys = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.System), file);
        if (File.Exists(sys)) return sys;
        var win = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Windows), file);
        return File.Exists(win) ? win : file;
    }

    /// <summary>
    /// Run a console tool and return its full stdout as one string (stderr ignored).
    /// For parsing tools like netsh where we want the whole output, not a live log.
    /// </summary>
    public static async Task<string> RunCaptureAsync(string exe, string args, int timeoutMs = 15000)
    {
        try
        {
            var psi = new ProcessStartInfo(ResolveSystemExe(exe), args)
            {
                UseShellExecute        = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow         = true,
            };
            using var p = Process.Start(psi)!;
            var stdout = p.StandardOutput.ReadToEndAsync();
            var exited = p.WaitForExitAsync();
            if (await Task.WhenAny(exited, Task.Delay(timeoutMs)) != exited)
                try { p.Kill(true); } catch { }

            // Bound the read as well. ReadToEndAsync only completes at pipe EOF, and a grandchild
            // that inherited the handle (dism.exe → DismHost.exe) keeps it open after the tool has
            // exited — so awaiting it unconditionally can hang a collector forever.
            return await Task.WhenAny(stdout, Task.Delay(DrainMs)) == stdout
                ? await stdout
                : string.Empty;
        }
        catch { return string.Empty; }
    }

    public static async Task<int> RunAsync(
        string exe, string args, Encoding? outputEncoding,
        Action<string> onLine, Action<int>? onProgress = null, CancellationToken cancel = default)
    {
        try
        {
            var psi = new ProcessStartInfo(ResolveSystemExe(exe), args)
            {
                UseShellExecute        = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow         = true,
            };
            if (outputEncoding != null)
            {
                psi.StandardOutputEncoding = outputEncoding;
                psi.StandardErrorEncoding  = outputEncoding;
            }

            using var p = Process.Start(psi)!;

            var exited     = new TaskCompletionSource();
            var stdoutDone = new TaskCompletionSource();
            var stderrDone = new TaskCompletionSource();
            bool abandoned = false;

            void Handle(string? data)
            {
                if (data == null || abandoned) return;
                // Tools redraw progress lines with \r — keep only the final segment.
                var seg = data.Contains('\r') ? data.Split('\r').Last() : data;
                seg = Ansi.Replace(seg, "").Replace("\0", "").Trim();
                if (seg.Length == 0) return;

                // DISM progress bar: "[====  42.0%  ====]" — and early frames like "[==   5.8%   ]"
                // have too few fill chars to trip a fill-count test, so bracket-wrapped counts too.
                var m = Percent.Match(seg);
                bool barish = (seg.StartsWith('[') && seg.EndsWith(']'))
                              || seg.Count(c => "=█░▒▓■□".Contains(c)) > 3;
                if (m.Success && barish)
                {
                    if (int.TryParse(m.Groups[1].Value, out var pct)) onProgress?.Invoke(pct);
                    return; // don't log the raw bar
                }
                // Lone spinner frame
                if (seg.Length <= 4 && seg.All(c => @"-|/\ ".Contains(c))) return;

                onLine(seg);
            }

            p.OutputDataReceived += (_, e) => { if (e.Data is null) stdoutDone.TrySetResult(); else Handle(e.Data); };
            p.ErrorDataReceived  += (_, e) => { if (e.Data is null) stderrDone.TrySetResult(); else Handle(e.Data); };
            p.EnableRaisingEvents = true;
            p.Exited += (_, _) => exited.TrySetResult();
            p.BeginOutputReadLine();
            p.BeginErrorReadLine();
            if (p.HasExited) exited.TrySetResult();   // could have raced us to the finish

            // Cancellation = kill the tree and let the normal Exited path finish the bookkeeping,
            // so a stopped run still drains, still audit-logs, and still returns an exit code.
            using var reg = cancel.Register(() => { try { p.Kill(true); } catch { } });

            // Wait for the PROCESS, not for the pipes to close. Process.WaitForExit[Async]() also
            // waits for stdout/stderr to hit EOF — and dism.exe leaves DismHost.exe children holding
            // the inherited pipe handles long after dism itself exits. Waiting on EOF therefore hangs
            // the card at "Running…" with the operation visibly finished in the log (0.19.7 bug).
            await exited.Task;

            // The process is gone and the exit code is final; give the readers a moment to flush the
            // last lines, then move on regardless of who is still holding the pipe. DISM always
            // leaves a DismHost.exe behind, so this window expiring is normal, not an error — don't
            // report it, just stop listening.
            var drained = Task.WhenAll(stdoutDone.Task, stderrDone.Task);
            if (await Task.WhenAny(drained, Task.Delay(DrainMs)) != drained)
                abandoned = true;   // stop logging into a card we've already reported as finished

            // Audit trail: every command the tool runs + its result (read-only collectors use
            // RunCaptureAsync, so this stays free of snapshot/query noise).
            ActivityLog.Command("cmd", exe, args, p.ExitCode);
            return p.ExitCode;
        }
        catch (Exception ex)
        {
            onLine($"Error: {ex.Message}");
            ActivityLog.Command("cmd", exe, args);
            ActivityLog.Result("cmd", $"failed to launch: {ex.Message}");
            return -1;
        }
    }
}
```
