---
project: PartnerTool
tags: [partnertool, deep-dive, wpf]
---

# WPF Single-File Quirks (hard-won)

## 1. Shutdown-telemetry crash on every close

**Symptom:** closing the app → "Something went wrong" dialog + WER `Application Error` events (`KERNELBASE.dll`, `0xe0434352`).
**Cause:** WPF's shutdown telemetry (`ControlsTraceLogger.LogUsedControlsDetails` inside `Application.CriticalShutdown`) does a name-based `Assembly.Load("System.Diagnostics.Tracing")` which **fails in a single-file publish** (no DLL on disk). Dev builds are unaffected.
**Failed fix:** `AppDomain.AssemblyResolve` hook (0.17.30) — *looked* fixed but Event Viewer later proved it fired on every close of every build; the resolver isn't consulted on that load path.
**Fix attempt (0.17.52):** `MainWindow.OnClosed` → `Environment.Exit(0)` — exit before CriticalShutdown's telemetry can run. Stopped the telemetry `FileNotFoundException`… but exposed a **second** exit crash.
**Part 2 (found 0.17.59):** WER events + `PartnerTool_errors.log` showed v0.17.55 *still* faulting on close, now with `DllNotFoundException` in the **native C++ CRT teardown** (`_app_exit_callback` → `__scrt_uninitialize_type_info` → `ModuleUninitializer.SingletonDomainUnload`). `Environment.Exit` still runs native module-unload callbacks, and a single-file-extracted native lib can already be gone by then. Intermittent (once per few sessions).
**Real fix (0.17.59):** `MainWindow.OnClosed` → `TerminateProcess(GetCurrentProcess(), 0)` (kernel32 P/Invoke). Ends the process immediately, skipping **all** teardown — managed finalizers, native dtors, atexit — so no exit-time fault is possible. Safe because every log/setting is written synchronously; nothing needs graceful shutdown. (A hard-terminate won't unload a loaded WinRing0 sensor driver, but that persists harmlessly until reboot and is usually HVCI-blocked here anyway.)
**Lesson:** verify "fixed" against Event Viewer over multiple days, not one session — and remember `Environment.Exit` is NOT the same as terminating: it still runs teardown that can throw. For a single-file app that needs no cleanup, `TerminateProcess` is the only guaranteed-clean exit.

## 2. WPF binding only sees **properties**

Public *fields* on a view-model bind silently to nothing (blank names, dead triggers, empty bars) — no error anywhere. Burned us on `UsageEntry` in Disk Usage. Always `{ get; init; }`/`{ get; set; }`.

## 2b. Local value beats a trigger Setter (placeholder bug)

The Installed-Software search box had a placeholder implemented as a Style `Trigger` on `Text=""` setting a `VisualBrush` background. It never showed — the TextBox also had an inline `Background="Transparent"`, and in WPF's dependency-property precedence a **local value outranks a style-trigger setter**. Fix (0.17.55): set the default `Background` via a Style `Setter` (not a local attribute) so the trigger can override it. Rule: anything a trigger must change cannot also be set as a local attribute.

## 2c. Auto-refresh DispatcherTimer never firing

System Info's "Refreshed at … updates every 1 minute" never changed. The timer used the parameterless `DispatcherTimer()` (**Background** priority) and was started only inside `IsVisibleChanged` — fragile. Fix (0.17.55): create it with **`DispatcherPriority.Normal`** and `Start()` it in the constructor for the life of the page, gating the *work* on `IsVisible` inside the tick instead of start/stopping the timer; wrap the refresh in try/catch so one collector fault can't silently freeze the loop. Interval is now 45 s. Lesson: for a timer that *must* run, don't hinge it on a visibility event — always-on + visibility-gated work is more robust.

## 3. Wi-Fi SSID needs Location permission on Win11

`netsh wlan show interfaces` (SSID/signal) is gated behind Location services **and** "Let desktop apps access your location" (`HKCU\...\ConsentStore\location\NonPackaged`). Registry writes do **not** enable it — only the Settings UI does. Empirically proven; we gave up on SSID and show only connected/not-connected via `NetworkInterface` (`Wireless80211` + `Up`), which needs no permission.

## 4. Other single-file facts

- A `Resources` subfolder relocation of the multi-file build breaks coreclr init — dead end, don't retry.
- LibreHardwareMonitor's WinRing0 driver is blocked by HVCI/vulnerable-driver blocklist on many Win11 machines → temps often unavailable; degrade gracefully.
- `sfc.exe` writes UTF-16 to stdout — decode `Encoding.Unicode` or the output is garbage.
- Defender ASR may block unsigned single-file exes; expect an unblock step on hardened endpoints.

## 6. Black window in hidden sessions (NinjaOne Backstage) — UNFIXABLE for WPF (proven 0.19.8)

**Symptom:** in NinjaOne Background Mode ("Backstage" — SYSTEM on a separate hidden desktop, no display device) the app launches but shows a black rectangle. GDI apps (regedit, Task Manager) display fine.

**What we proved with a live five-mode probe (BackstageProbe, session of 2026-07-14):**
- A: normal WPF, hardware rendering → black
- B: per-window `RenderMode.SoftwareOnly` → black
- v5: process-wide `ProcessRenderMode = SoftwareOnly` → black
- C: layered window (`WindowStyle=None` + `AllowsTransparency`) → black
- D: WPF → offscreen `RenderTargetBitmap` → GDI `BitBlt` in a subclassed `WM_PAINT` (wndproc replaced via `SetWindowLongPtr` so it runs BEFORE WPF's handler) → **bitmap is empty: px=0% in every mode**. WPF's render thread never comes up on that desktop; it cannot rasterize even offscreen.
- E: pure GDI (`FillRect` + `DrawText` in `WM_PAINT`) → **displays perfectly** (the regedit path).

**Conclusion:** only GDI-drawn UIs (Win32, WinForms, console) can display in Backstage. No WPF-side trick — render mode, layered windows, offscreen render + GDI blit — can work, because the failure is at **rasterization**, not presentation.

**Debug trick that made this diagnosable:** window TITLE BARS render via non-client GDI and are visible in Backstage — the probe reported live counters (`paints=N err=N px=N%`) in its titles, giving telemetry through a capture that showed only black content.

**What shipped (0.19.8):** the session-mismatch software-rendering fallback + `-softwarerender` switch + startup breadcrumbs in `PartnerTool_errors.log` stay — correct for RDP/odd-GPU cases, free on normal desktops — but Backstage support would require a GDI-based companion UI (WinForms panel or console menu).

**Expected Backstage caveats regardless:** winget = "unavailable to this account" (SYSTEM); `ms-settings:`/Control Panel links won't open (no Explorer there).