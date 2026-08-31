---
project: PartnerTool
tags: [partnertool, disaster-recovery, bootstrap]
---

# New Machine Bootstrap

Fresh PC + restored vault backup → productive again. Claude's persistent memory is **per-machine**
(it lives in `%USERPROFILE%\.claude\`, not in this vault), so on a new machine a fresh session
knows nothing until pointed here. This note is the pointer.

## 1. First Claude session — paste this

> This project's knowledge base is my Obsidian vault at `C:\Obsidian Vaults\Vibe Projects`
> (adjust path if restored elsewhere). Read `01 Projects\PartnerTool\PartnerTool.md` (the hub) and
> `Conventions & Security Model.md` before doing anything. Then save the key conventions to your
> memory (vault location, versioning ritual, logging rule, no-wmic, American English, elevated
> security model) so future sessions load them automatically.

## 2. Install prerequisites

- **Windows 11**, admin account.
- **git** — `winget install Git.Git` (the code now lives on GitHub; needed to clone/pull/push).
- **.NET 10 SDK** — `winget install Microsoft.DotNet.SDK.10` (the only hard requirement to build).
- **WiX v5 tool** — only if building the MSI: `dotnet tool install --global wix --version 5.0.2`.
  **Pin v5** — it builds the v4-schema `Product.wxs` fine, and **v6+ carries the Open Source
  Maintenance Fee** for commercial use (learned on the [[Claude Dev VM]]). The single-file exe needs
  nothing beyond the SDK.
- Obsidian itself is optional — the vault is plain Markdown; Claude reads it directly.

## 3. Get the repo

**Primary (the code is now on git/GitHub):**

```powershell
git clone https://github.com/N3cTr0/PartnerTool.git C:\Projects\PartnerTool
```

Private repo `N3cTr0/PartnerTool`, `main` branch — you'll need GitHub auth (a PAT or the git
credential manager) the first time. `bin/obj/dist/release/.vs` are git-ignored, so a fresh clone is
source-only; build per [[Build & Release]].

**One-time setup in the clone — the friction we actually hit on this VM (do these or they bite):**

```powershell
$repo = "C:\Projects\PartnerTool"
$git  = "C:\Program Files\Git\cmd\git.exe"
# 1. Git identity — else the first commit dies with "Author identity unknown".
& $git -C $repo config user.email "lowe.gc25@gmail.com"
& $git -C $repo config user.name  "N3cTr0"
# 2. Enable the vault auto-sync post-commit hook (per-clone; NOT carried in the repo).
& $git -C $repo config core.hooksPath .githooks
```

- **NuGet:** a bare VM has an empty NuGet config, so `dotnet build/publish` fails with **NU1101**
  (can't find LibreHardwareMonitorLib / the runtime packs). The repo now ships a `nuget.config`
  pinning nuget.org, so restore just works. Note this covers builds *inside* the repo only - a bare
  user-level NuGet config still breaks `dotnet tool install --global` (see the section below).
- **GitHub PAT (check this first):** the credential is a fine-grained PAT covering **both** `PartnerTool` and `vibe-vault`
  (the vault backup), so when it lapses the code push and the vault backup stop together. The
  2026-08-17 expiry recorded here was never confirmed - pushes still worked on 2026-08-30, so treat
  any date as unverified and just confirm a push early (see the section below).
- **`gh` CLI** (optional): not installed by default — `winget install GitHub.CLI` + `gh auth login`
  only if you need to create repos from the VM.

**Offline fallback** (no network / GitHub access): [[Restore From Snapshot]] rebuilds the entire
source tree from the vault's `Code\` folder (round-trip re-tested 18 Jul 2026 at v0.19.14, all
files content-clean), and `Code\_Assets\` has the binary logo files. Then build per [[Build & Release]].

## 4. Paths & environment notes

- **Standard dev path is now `C:\Projects\PartnerTool`** on the [[Claude Dev VM]] (the sole dev
  environment). The offline restore script hardcodes it; `tools\sync-vault.ps1` is path-portable
  (`-VaultPath` / `$env:PARTNERTOOL_VAULT`). Only touch paths if you ever clone elsewhere.
- `C:\PCI\…` (logs/tools) is created and ACL-hardened by the app itself on first run — nothing to do.
- Defender/ASR may block the freshly built exe (new hash, zero prevalence) — see the confirmed
  ASR caveat in [[Build & Release]].

## Moving off the dev VM (Aug 2026)

Handover state at **v0.24.24**, the point the project left the VM for a physical PC:

- Both repos are pushed and clean - `N3cTr0/PartnerTool` and `N3cTr0/vibe-vault`, branch `main`.
- The vault `Code\` snapshot is current at v0.24.24 and round-trip verified ([[Restore From Snapshot]]).
- Nothing else on the VM is needed: `dist\` rebuilds, and `C:\PCI\` is recreated by the app on
  first run. The old machine's `C:\PCI\Logs` audit trail is the only thing that does not travel -
  copy it off separately if any of it still matters.

**Do these two first on the new PC:**

1. **Check the GitHub PAT.** One fine-grained PAT covers both repos, so when it lapses the code push
   and the vault backup stop together - which reads like the sync breaking rather than an auth
   problem. A date of 2026-08-17 was once recorded, but pushes still worked on 2026-08-30, so treat
   any expiry date as unverified and just confirm a push works early. Note a single failure can be
   transient: one push was rejected with "Invalid username or token" and an immediate retry
   succeeded.
2. **Re-point Claude at the vault** using the paste in section 1 - a fresh session on a new machine
   knows nothing about this project until you do.

**Version-bump pitfall carried over from the VM:** bumping the version in `PartnerTool.csproj` and
`installer/Product.wxs` with PowerShell `Set-Content -Encoding utf8` writes a **UTF-8 BOM**. It did
exactly that to `Product.wxs`, which had been BOM-less for its whole history. Nothing broke - MSBuild
and WiX both accept it - but write those two files with `[IO.File]::WriteAllText` and a
`UTF8Encoding($false)`, or a byte-level diff against the vault snapshot will show phantom changes.

## What the move actually took (verified 08/30/2026)

The vault and both repos came across intact - repo clean at v0.24.24 matching `origin/main`, the
`Code\` snapshot already current, and the PAT still authenticating to both repos. Four things still
needed doing on the bare PC, in this order:

1. **Everything copied off the VM carried Mark of the Web.** All 174 repo files (and 76 vault notes)
   had a `Zone.Identifier` stream, so Windows treated them as "came from another computer".
   `dotnet build` tolerated it, but **`dotnet tool restore` refused to read
   `.config\dotnet-tools.json`** - which breaks `build-installer.bat` at step 1. Fix:
   `Get-ChildItem -Recurse -File -Force | Unblock-File` over both trees. It strips only the alternate
   data stream, so file content is untouched and git stays clean.
2. **The user-level NuGet config had an empty `<packageSources>`.** `%APPDATA%\NuGet\nuget.config`
   did not travel with the home directory. Project builds still worked, because the repo's own
   `nuget.config` uses `<clear/>` + nuget.org - but anything run *outside* the repo, including
   `dotnet tool install --global`, failed with "No NuGet sources are defined or enabled". Fix:
   `dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org`.
3. **WiX was not installed** - no global tool, and the local manifest could not restore until (1) and
   (2) were fixed. `dotnet tool restore` now pulls the pinned 5.0.2. It suggests upgrading to 7.0.0 on
   every restore; **ignore that** - v6+ carries the Open Source Maintenance Fee.
4. **Do not assume the Claude session is elevated.** It ran with a *filtered* token here
   (Administrators filtered out, `IsInRole(Administrator)` = False), so `tools\ui-drive.ps1` and
   launching the `requireAdministrator` exe both need an explicitly elevated session. Check rather
   than assume - this has differed between sessions on this machine.

Verified working afterwards: `dotnet build -c Release` clean, `build-installer.bat` end to end
(single-file exe + MSI, both stamped 0.24.24), `tools\sync-vault.ps1` (133 code notes), and
fetch/push auth against both `N3cTr0/PartnerTool` and `N3cTr0/vibe-vault`.
## What transfers vs. what rebuilds

| Thing | Transfers in the vault backup? |
|---|---|
| All project knowledge, conventions, lessons, changelog | ✔ yes |
| Full source code + logo assets | ✔ yes (Code snapshot) |
| Built exes / obj / bin | ✘ rebuild (one publish command) |
| Claude's memory index | ✘ rebuilds itself from this vault via step 1 |
| `C:\PCI` logs from the old machine | ✘ not in the vault — back up separately if the audit trail matters |