---
project: Octavia Android
tags: [octavia, octavia-android, build]
---

# Build & Release

> How to build the app, get it onto a device, and talk to her from it. Everything here works **without elevation and without Android Studio**, which was not the original assumption.

## The toolchain, all user-scope

Installed 08/31/2026 on `N3CTR0-PC` with `winget --scope user` — no UAC prompt at any point:

| Tool | Version | Where |
|---|---|---|
| Android SDK Platform-Tools (`adb`) | 37.0.1 | `%LOCALAPPDATA%\Microsoft\WinGet\Packages\Google.PlatformTools_*\platform-tools` |
| Microsoft OpenJDK 17 | 17.0.10 | `%LOCALAPPDATA%\Programs\Microsoft\jdk-17.0.10.7-hotspot` |
| scrcpy | 4.1 | mirror and control the device from the PC |
| **Google `android` CLI** | 1.0.15985488 | the one that made Studio unnecessary |
| Android SDK | platform 36 | `%LOCALAPPDATA%\Android\Sdk` |

**Android Studio is not needed.** The `android` CLI covers `create`, `sdk`, `emulator`, `run`, `install`, `screen` and `layout`. Install Studio anyway if the visual layout editor is wanted; nothing is blocked without it.

Two gotchas that cost time once each:

- **`--no-metrics` is a *global* flag** — `android --no-metrics sdk list`, not `android sdk list --no-metrics`. It reports usage data otherwise.
- **`winget` writes the new PATH to the registry**, so a shell started before the install will not see it. Rebuild PATH from `[Environment]::GetEnvironmentVariable('Path','Machine'/'User')` in the same call, or the tools look missing when they are not.
- **Temurin 17 has no user-scope installer.** Microsoft OpenJDK does, and covers it.

## Building

```
$env:JAVA_HOME = 'C:\Users\N3cTr0\AppData\Local\Programs\Microsoft\jdk-17.0.10.7-hotspot'
cd C:\Projects\Octavia-Android
.\gradlew.bat assembleDebug
```

First build pulls Gradle 9.1.0 and every dependency — several minutes and roughly 300 MB into `~\.gradle`. After that it is fast. The APK lands in `app\build\outputs\apk\debug\app-debug.apk`.

## Getting it onto the J7 Pro

```
adb install -r app\build\outputs\apk\debug\app-debug.apk
adb shell monkey -p com.n3ctr0.octavia -c android.intent.category.LAUNCHER 1
```

The device must be unlocked and USB debugging authorised — `adb devices` shows `unauthorized` until the prompt on the handset is accepted.

## The development loop, which needs nothing exposed

```
adb reverse tcp:8848 tcp:8848
```

**This is the important line.** It points the handset's own loopback at the host's, over USB. So:

- her socket stays bound to `127.0.0.1`,
- `RemoteAccess` stays **off**,
- **no Windows firewall rule** is needed,
- **no elevation** is needed,
- and because the connection arrives at her *from loopback*, the **per-run token works** — no pairing with the durable remote key required just to build.

Wireguard and the remote key replace only the address, later. The whole of [[Roadmap]] stage 1 was proven this way.

Her per-run token is in `C:\Projects\Octavia\data\octavia.log`:

```
face socket listening on ws://127.0.0.1:8848/?token=<token>
```

## Seeing what the app is doing

```
adb exec-out screencap -p > shot.png      # a still
scrcpy                                     # live mirror, and you can drive it
adb logcat -s AndroidRuntime:E             # crashes only
```

Screenshots for a release go in `Screenshots\`, named `v<version> - <what it shows>.png`, which is what `tools\check-vault.ps1` looks for.

### Driving the UI from here

`adb shell input tap <x> <y>` and `input text <s>` work, with one trap: **a loop of `adb shell` calls is not a sequence of keystrokes.** Each is a fresh round trip against a device that is also busy rendering, and presses get dropped — clearing a field with forty separate `KEYCODE_DEL` calls landed fifteen of them, leaving the tail of the old value glued to the new one.

Pass several keycodes to **one** invocation instead:

```
adb shell "input keyevent 67 67 67 67 …"     # 67 = KEYCODE_DEL
```

**For pairing, do not use the dialog at all.** The burst gets truncated — seventy `KEYCODE_DEL` deleted exactly fifteen characters, twice — so write the preferences file directly instead. This is the reliable way to pair during development:

```
adb push prefs.xml /data/local/tmp/ && adb shell chmod 644 /data/local/tmp/prefs.xml
adb shell am force-stop com.n3ctr0.octavia
adb shell run-as com.n3ctr0.octavia cp /data/local/tmp/prefs.xml shared_prefs/octavia.xml
```

`run-as … sh -c 'cat > …'` fails with *permission denied*; `cp` from a world-readable file works. Force-stop first, or the running process overwrites it on the way out.

And read back what actually landed rather than trusting the taps — a debuggable build allows:

```
adb shell run-as com.n3ctr0.octavia cat /data/data/com.n3ctr0.octavia/shared_prefs/octavia.xml
```

### Her token changes on every restart

The per-run token is regenerated each time she starts, so restarting her un-pairs the phone. That is what the durable **remote key** is for — but `data\remote.key` is created *lazily*, on first use, so it does not exist until something asks for it. During a session where she restarts often, expect to re-pair, and watch her log for `face socket refused a connection with a bad or missing token`.

## Versioning

Pre-release `0.x`, matching [[Changelog|hers]]. **PATCH by default**; MINOR for a new subsystem or a notable behaviour change. Roadmap stages are substantial by definition, so a completed stage takes a MINOR.

The version lives in `versions.md`'s newest header — both vault scripts read it from there, so it cannot drift from the release it documents.

## Vault upkeep

Run both, in the same change set as the code:

```
pwsh -NoProfile -ExecutionPolicy Bypass -File tools\sync-vault.ps1
pwsh -NoProfile -ExecutionPolicy Bypass -File tools\check-vault.ps1
```

`check-vault` exits with the number of problems, so it can gate a commit. **The execution-policy flags are required on this machine** — it blocks unsigned scripts, and the machine-wide policy is deliberately left alone.

## Links

- [[Octavia Android]] — the hub
- [[Architecture]] · [[Conventions & Security Model]] · [[Restore From Snapshot]]
- [[Moving To The New Machine]] — why elevation and execution policy behave as they do here
