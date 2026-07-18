---
project: PartnerTool
tags: [partnertool, build]
---

# Build & Release

## Dev build (fast iterate)

```powershell
dotnet build C:\Projects\PartnerTool\PartnerTool\PartnerTool.csproj -c Release -v minimal
```

Outputs to `bin\Release\<tfm>\`. If the exe is locked (`MSB3027`), the app is running — close it first (`Get-Process PartnerTool`).

## Single-file distribution build (what techs get)

```powershell
dotnet publish PartnerTool.csproj -c Release -r win-x64 --self-contained true `
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true `
  -p:EnableCompressionInSingleFile=true -p:DebugType=none `
  -p:AppendRuntimeIdentifierToOutputPath=false -o ..\dist
Rename-Item ..\dist\PartnerTool.exe "PartnerTool-<version>.exe"
```

- ~66 MB, everything embedded. Rename with the version so `dist\` history is self-describing.
- **Verify 0 warnings / 0 errors** before handing out.

## MSI (optional)

The **v4-schema** `installer\Product.wxs` (built with the **WiX v5** tool — pin `--version 5.0.2`, since v6+ carries the Open Source Maintenance Fee; see [[New Machine Bootstrap]]) → installs a folder build to `C:\PCI\PartnerTool`, per-machine, MajorUpgrade so reinstalling updates in place. Harvests `release\**` (a normal folder publish, *not* single-file).

## Versioning ritual (every change)

1. Bump `<Version>` in `PartnerTool.csproj`
2. Bump `Version` in `installer\Product.wxs`
3. Add a newest-first entry in `versions.md` (mirrored to [[Changelog]])
4. Commit + push to GitHub (see below)
5. Vault snapshot (`Code\`, `_Code Index`, [[Changelog]]) auto-regenerates via the **post-commit hook** (`tools\sync-vault.ps1`) — no manual step

## Source control (git → GitHub, since 2026-07-18)

The repo is under **git**, remote **private GitHub `N3cTr0/PartnerTool`** (`main`). `git.exe` is at
`C:\Program Files\Git\cmd\git.exe` (not always on PATH — call the full path or add it). `.gitignore`
excludes `bin/ obj/ dist/ release/ .vs/ *.user *.suo` and WiX intermediates, so commits are source-only
(the ~66 MB built exes are **not** in git — they live in `dist\` locally and are handed out directly).

```powershell
$git = "C:\Program Files\Git\cmd\git.exe"
& $git -C C:\Projects\PartnerTool add -A
& $git -C C:\Projects\PartnerTool commit -m "0.19.x: <what changed>"
& $git -C C:\Projects\PartnerTool push
```

Commit after the version bump so the commit message and `versions.md` entry agree. GitHub is the
primary source-of-truth; the vault `Code\` snapshot ([[Restore From Snapshot]]) is the offline backup.

## Known distribution caveats

- **Confirmed (7 Jul 2026):** the ASR rule *"Block executable files from running unless they meet a
  prevalence, age, or trusted list criterion"* (`01443614-CD74-433A-B99E-2ECDC07BFC25`, pushed via
  Intune/MDE) blocks fresh PartnerTool builds — every build is a new unsigned zero-prevalence hash.
  Error shown: "Windows cannot access the specified device, path, or file." Evidence: Defender
  Operational log event **1121**. Remediation: per-rule ASR exclusion for `C:\PCI\PartnerTool\*`
  in Intune (fleet), `Add-MpPreference -AttackSurfaceReductionOnlyExclusions <path>` elevated
  (single machine, may be reverted by policy sync); durable fix = **code-sign the exe**, which
  satisfies the rule's trusted-list criterion.
- A `Resources` subfolder relocation for the multi-file build is a **verified dead end** (breaks coreclr init).
- LibreHardwareMonitor temps need the WinRing0 driver — blocked by HVCI/memory-integrity on many Win11 boxes; the app degrades gracefully.
