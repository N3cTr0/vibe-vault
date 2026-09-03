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

## The desktop shortcuts

**Two of them since v0.26.0**, and the order matters — the server has to be up before the client has anything to attach to.

| Shortcut | Runs | Arguments |
|---|---|---|
| `Octavia Server.lnk` | `Octavia.Server.exe` | `--profile home` |
| `Octavia.lnk` | `Octavia.exe` (the client) | *(none)* |
| `Octavia Controls.lnk` | `Octavia.Control.exe` | *(none)* — **the one to put in Startup** |
| `Start Octavia.lnk` / `Stop Octavia.lnk` | `Octavia.Server.exe` | `--start` / `--stop` |

**`Octavia Controls.lnk` is the useful one to keep**, since v0.46.0: the start/stop pair still work, but the controls also say whether she is running and let a setting be changed while she is not. See [[Her Controls]].

Made by `tools\make-shortcuts.ps1`, which overwrites, so running it twice is safe. Re-run it after moving the repo or changing which rig the icons should name.

```
pwsh -NoProfile -ExecutionPolicy Bypass -File tools\make-shortcuts.ps1
pwsh -NoProfile -ExecutionPolicy Bypass -File tools\make-shortcuts.ps1 -ProfileName cloud
pwsh -NoProfile -ExecutionPolicy Bypass -File tools\make-shortcuts.ps1 -Dist -Minimised
```

*It is a script rather than a manual step because the paths here have cost time twice.* The Desktop is OneDrive-redirected on this machine, so `C:\Users\N3cTr0\Desktop` **does not exist** — take it from `[Environment]::GetFolderPath('Desktop')`. And the client's *assembly* is `Octavia.exe` while its project is `Octavia.App`, which is also why `InternalsVisibleTo` needed the former.

Three deliberate choices:

- **They point at the Debug build, not `dist`.** The Debug exes are rewritten by every build, so the icons are always the latest Octavia and can never go stale. `dist` is for handing to someone else. `-Dist` switches them.
- **The server names its profile.** Without `--profile` it inherits whatever `config.json` says, which she rewrites herself — so the icon's brain could change without anyone touching the icon. See [[Profiles & Configuration]]. A second shortcut with `--profile cloud` keeps both brains one double-click away.
- **The client does not.** A profile is a brain, a Whisper model and a set of tool servers, and the client has none of those.

> **The split silently broke the old shortcut**, and it is worth recording as a shape rather than an incident. `Octavia.lnk` had pointed at the client exe with `--profile dev` since the machine move — after v0.26.0 that argument reached a process which does not parse it, and the icon opened a client with no server to talk to. Nothing errored. **An argument that stops being understood is not an error, it is silence**, and a shortcut is exactly the place nobody re-reads. Hence the script.

The server's console window opens normally rather than minimised, because the first thing anybody does with a new server is watch it start — and that window is also how you stop it. `-Minimised` for an always-on arrangement.

The server exe wears her icon too, borrowed from the client's assets rather than copied: it is *her* mark, not either executable's, and a generic console icon beside an Octavia one reads as two unrelated programs.

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

**Four publishes since v0.46.0**, in this order:

```
rmdir /s /q C:\Projects\Octavia\dist
dotnet publish src\Octavia.Kokoro -c Release -r win-x64 --self-contained false -o C:\Projects\Octavia\dist
dotnet publish src\Octavia.Server -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:PublishReadyToRun=true -o C:\Projects\Octavia\dist
dotnet publish src\Octavia.App -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:PublishReadyToRun=true -o C:\Projects\Octavia\dist
dotnet publish src\Octavia.Control -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:PublishReadyToRun=true -o C:\Projects\Octavia\dist
```

Produces `dist\Octavia.Server.exe`, `dist\Octavia.exe`, `dist\Octavia.Control.exe` and `dist\octavia-kokoro.exe`, plus one `wwwroot` folder beside them. **All of it must travel.** The face is deliberately left outside the exes so it can be edited on the target machine without a rebuild.

| Left out | What happens |
|---|---|
| `octavia-kokoro.exe` | She runs and **cannot speak**. Her self-test says *"octavia-kokoro.exe is not installed"* rather than reporting a missing download |
| `Octavia.Control.exe` | She runs, and there is no tray icon and nothing to change a setting with but a text editor — see [[Her Controls]] |

> **Kokoro first, and framework-dependent.** Self-contained would put a second copy of the .NET runtime beside the one the server already carries, for an executable whose whole job is to hold a model — and publishing it first means the two self-contained publishes after it own every file they share.

> **Then the server, then the client.** Both of those carry `Octavia.Core`, so today the order only decides which identical copy wins — but publishing the client last would leave the *client's* `wwwroot` in place, and that stops being harmless the moment one of them stops shipping the page.

**Only the server needs to go on the machine she lives on.** A client is small and needs nothing but a network route and the remote key — see [[A Server, And Clients]].

> **One version covers all three assemblies**, in `Directory.Build.props`. It used to live in `Octavia.App.csproj`, which was also the only project; three copies kept in step by hand would disagree in a diagnostics bundle before anybody noticed, because `SystemReport.Version` reads whichever assembly happens to be executing. `tools\shoot.ps1`, `sync-vault.ps1` and `check-vault.ps1` all read it from there now, and `EarsTest -- split` fails if a project quietly grows its own.

Drop `--self-contained true` and the two `PublishSingleFile` switches if the target already has the **.NET 10 Desktop Runtime** — a few MB instead.

**Never add `PublishTrimmed`.** WPF, `System.Speech` and the JSON serialiser all resolve types by reflection; it publishes cleanly and dies at startup.

## Moving her to another PC

Copy `Octavia.exe` and `wwwroot`. Then the three things that do not travel:

- **The API key.** DPAPI-sealed to one account on one machine — see [[Conventions & Security Model]]. Paste it in again; do not copy `apikey.dat`.
- **Config.** Lives in `<data>\config.json`, not beside the exe. Copy it by hand to carry a tuned profile across.
- **Whisper models.** Re-downloaded on first listen into `<data>\models`.

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

**Currently manual.** Git arrived on 08/30/2026 but the post-commit hook has not been wired yet — do it the way PartnerTool does (`core.hooksPath .githooks`, set per-clone).

`wwwroot\lib\` is excluded as vendored third-party code — three.js and three-vrm would otherwise add 3 MB of somebody else's source to the vault on every sync. `Screenshots\` is excluded too: it is evidence, not source — see [[Screenshots]].

Since v0.6.0 the script also **regenerates the project notes that mirror a repo document** — [[Changelog]], [[Roadmap]] and [[Face Protocol]]. They were hand-copied before that and drifted every time the repo changed, which is the one failure a snapshot tool must not have.

Run it with `-PruneOrphans` after renaming or deleting a source file; the plain run reports orphans but never deletes them. It caught `VoiceBox.cs` after the rename to `SapiVoice.cs`.

## Not yet

- ~~No git repo and no remote.~~ **Resolved 08/30/2026:** git history plus a private GitHub remote at [`N3cTr0/Octavia`](https://github.com/N3cTr0/Octavia). The vault snapshot is now the second copy rather than the only one — still worth keeping current, since it is the readable one and it is what [[Restore From Snapshot]] is written against.
- No installer. The single exe is the distribution.
