---
project: PartnerTool
tags: [partnertool, deep-dive, security]
---

# Tech Gate & Activity Log

The two halves of "a tech ran something on a production box — what happened?"

## Tech Gate (`TechGate.cs`, `TechGateWindow`)

- **Code = `day + year`** of today's date: 26 Jun 2026 → `262026`. Known to tech leads, not printed anywhere in the UI.
- Prompt deliberately doesn't hint it's date-based: *"This action can change or remove data on this machine. / Enter tech code to continue."* Input is a **PasswordBox** (masked) so remote viewers can't read it.
- **Unlock lasts 15 idle minutes (was 20 in 0.19.7; user tightened it in 0.19.10)** (`TechGate.Timeout`, since 0.19.7). First gated action verifies; later gated actions pass straight through *and slide the window forward*, so an actively-working tech is never re-prompted mid-job. Techs leave the app (often the Update All window) open and unattended on a client's desk — a process-lifetime unlock handed the machine to whoever sat down next. On a re-ask after expiry the prompt says *"Tech verification expired after 15 minutes idle"* so it doesn't read as a bug. Wrong code → "Incorrect code. Ask the tech lead if you're not sure."
- State is `_verifiedUntil` (UTC deadline), not a bool. `TechGate.IsUnlocked` / `TechGate.Remaining` to query, `TechGate.Lock()` to drop it, `TechGate.Changed` to be told when it flips.
- **Header lock indicator** (`MainWindow.BtnGate`, 0.19.7): grey closed padlock **Locked**, or amber open padlock + live **mm:ss countdown**. Click while unlocked → `Lock()` (step-away button); click while locked → opens the code prompt to unlock ahead of time. A `DispatcherTimer` ticks it every second, but `PaintGate` short-circuits when locked-and-already-showing-it, so the idle case does no work. Both transitions hit the activity log (`TechGate` category).
- Pattern in handlers: `if (!TechGate.Verify(Window.GetWindow(this))) return;` **before** the confirm dialog.

## Activity Log (`ActivityLog.cs`)

Append-only audit trail: `C:\PCI\Logs\PartnerTool_activity.log`, session header (`===== Session <time> · MACHINE\user · PartnerTool <ver> =====`).

- `ActivityLog.Command(cat, exe, args, exit)` — **auto-called by `ProcessRunner.RunAsync`** → every CLI command + exit code is recorded for free. `RunCaptureAsync` deliberately does *not* log (read-only collector queries would flood it).
- `ActivityLog.Action(cat, what)` + `Result(cat, outcome)` — called explicitly by any system change that doesn't shell out (registry/WMI/file ops): service control, profile delete, hosts save, startup toggles, power actions, kill process, WPAD toggle, temp clean summary, installer cleanup, sensitive reveals (BitLocker key, Wi-Fi password).

Per-fix logs also land in `C:\PCI\Logs\PartnerTool_<Tag>_<machine>_<stamp>.log` (each Repair card writes its inline log there via `SaveLog`), and app crashes go to `PartnerTool_errors.log`.

## Rule for new code

New CLI action → route through `ProcessRunner.RunAsync` (never bare `Process.Start` for system changes). New non-CLI change → add `Action`/`Result` calls. New destructive UI → add the TechGate check. See [[Conventions & Security Model]].
