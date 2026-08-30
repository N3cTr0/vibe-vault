---
project: Octavia
tags: [octavia, build]
---

# Build & Release

## Dev build

```
dotnet run --project src\Octavia.App
```

Runs from `src\Octavia.App\bin\Debug\net10.0-windows10.0.19041.0\Octavia.exe`. She hides to the tray on close — quit from the tray menu, or the next build fails with a file lock (`MSB3027`, "the file is locked by: Octavia").

## The desktop shortcut

`C:\Users\Claude\Desktop\Octavia.lnk` → that same **Debug** exe, with arguments `--profile dev`.

Two deliberate choices:

- **It points at the Debug build, not `dist`.** The Debug exe is rewritten by every build, so the shortcut is always the latest Octavia and can never go stale. `dist` is for handing to someone else.
- **It names its profile.** Without `--profile dev` the shortcut inherits whatever `config.json` says, which the app itself rewrites — so the icon's brain could change without anyone touching the icon. See [[Profiles & Configuration]].

A second shortcut with `--profile live` is the way to keep both brains one double-click away.

## Command line

| Argument | Does |
|---|---|
| `--profile <name>` (`-p`, `--profile=<name>`) | Pin the rig — see [[Profiles & Configuration]] |
| `--diagnostics <path>` | Write a diagnostics bundle with no window and no session, then exit |

`--diagnostics` runs **before** the single-instance check, so it works while she is running and, more to the point, when she will not start at all. See [[Diagnostics]].

## Test harness

```
dotnet run --project tools\EarsTest                # full suite
dotnet run --project tools\EarsTest -- mic         # capture-device diagnostic
dotnet run --project tools\EarsTest -- music       # what she makes of what is playing
dotnet run --project tools\EarsTest -- music demo  # ...while playing a known tempo
dotnet run --project tools\EarsTest -- beats       # beat detection only, no device needed
dotnet run --project tools\EarsTest -- small.en    # test against a specific Whisper model
```

The suite synthesizes speech, runs it through Silero VAD and Whisper, asserts silence produces nothing, exercises the streaming `<think>` filter and markdown flattener, checks config-profile precedence and the face protocol, and probes whatever local model server is configured. Exit code is the failure count.

The config checks write to a temporary file via `OCTAVIA_CONFIG`, so running the suite never disturbs the real settings.

## Developing the face

```
pwsh -File tools\serve-face.ps1        # http://127.0.0.1:8999/index.html
```

Serves `wwwroot` over loopback so the renderer can be opened in an ordinary browser, with devtools and `window.Face` on the console:

```js
Face.setMusic({ playing: true, bpm: 128, energy: 0.8 })
setInterval(() => Face.setMusic({ beat: true }), 60000 / 128)
```

Nothing else works — there is no host, so every control is inert and it says so. That is the point: this is for the renderer, not the app. Inside WebView2 a change costs a rebuild, a relaunch and a screenshot; here it costs a refresh.

A face served this way offers [[The Dev Panel]] without being told to, since a face on its own is being worked on by definition.

One trap when driving it from an automated browser: **`requestAnimationFrame` does not run in a hidden tab**, so anything the render loop drives reads as its initial value. Screenshots still work, because capture forces a paint — which makes the wrong measurement look believable. Check `document.hidden` first.

## Publish

```
dotnet publish src\Octavia.App -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:PublishReadyToRun=true -o C:\Projects\Octavia\dist
```

Produces `dist\Octavia.exe` (~310 MB, .NET runtime included) plus a `wwwroot` folder beside it. **Both must travel.** The face is deliberately left outside the exe so it can be edited on the target machine without a rebuild.

Drop `--self-contained true` and the two `PublishSingleFile` switches if the target already has the **.NET 10 Desktop Runtime** — a few MB instead.

**Never add `PublishTrimmed`.** WPF, `System.Speech` and the JSON serialiser all resolve types by reflection; it publishes cleanly and dies at startup.

## Moving her to another PC

Copy `Octavia.exe` and `wwwroot`. Then the three things that do not travel:

- **The API key.** DPAPI-sealed to one account on one machine — see [[Conventions & Security Model]]. Paste it in again; do not copy `apikey.dat`.
- **Config.** Lives in `%APPDATA%\Octavia\config.json`, not beside the exe. Copy it by hand to carry a tuned profile across.
- **Whisper models.** Re-downloaded on first listen into `%APPDATA%\Octavia\models`.

The target also needs the **WebView2 runtime** (present on Windows 11 by default; she names it in a message if missing) and, for the `dev` profile, a running local model server.

## Versioning

Bump `src\Octavia.App\Octavia.App.csproj` `<Version>` and `versions.md` together. PATCH by default; MINOR for a completed roadmap stage or a new subsystem. See [[Conventions & Security Model]].

## Vault sync

**Standing rule: the vault is updated in the same change set as the code**, not as an occasional catch-up. Two scripts:

```
pwsh -File tools\sync-vault.ps1                 # regenerate Code\, _Code Index, Changelog
pwsh -File tools\sync-vault.ps1 -PruneOrphans   # also delete notes whose source is gone
pwsh -File tools\check-vault.ps1                # is it current, and does it still restore?
```

`check-vault.ps1` answers both questions that matter after a change: is any source file newer than the snapshot, and does every note still reproduce its source byte-for-byte. Exit code is the problem count, so it can gate a commit once git exists. Run it *after* syncing.

Remember the hand-written half — `Features\`, `Deep Dives\`, [[Lessons Learned]], [[Roadmap]] — does not sync itself and rots silently.

**Currently manual** — there is no git repo yet, so no post-commit hook. When git arrives, wire it the way PartnerTool does (`core.hooksPath .githooks`, set per-clone).

`wwwroot\lib\` is excluded as vendored third-party code — three.js and three-vrm would otherwise add 3 MB of somebody else's source to the vault on every sync. `Screenshots\` is excluded too: it is evidence, not source — see [[Screenshots]].

Since v0.6.0 the script also **regenerates the project notes that mirror a repo document** — [[Changelog]], [[Roadmap]] and [[Face Protocol]]. They were hand-copied before that and drifted every time the repo changed, which is the one failure a snapshot tool must not have.

Run it with `-PruneOrphans` after renaming or deleting a source file; the plain run reports orphans but never deletes them. It caught `VoiceBox.cs` after the rename to `SapiVoice.cs`.

## Not yet

- No git repo and no remote. **The vault snapshot is currently the only off-repo copy of the source.**
- No installer. The single exe is the distribution.
