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
git clone https://github.com/N3cTr0/PartnerTool.git C:\Users\graemel\Projects\PartnerTool
```

Private repo `N3cTr0/PartnerTool`, `main` branch — you'll need GitHub auth (a PAT or the git
credential manager) the first time. `bin/obj/dist/release/.vs` are git-ignored, so a fresh clone is
source-only; build per [[Build & Release]].

**Offline fallback** (no network / GitHub access): [[Restore From Snapshot]] rebuilds the entire
source tree from the vault's `Code\` folder (round-trip re-tested 18 Jul 2026 at v0.19.14, all
files content-clean), and `Code\_Assets\` has the binary logo files. Then build per [[Build & Release]].

## 4. Paths that may need adjusting on a new machine

- The restore script and snapshot-regeneration script hardcode the vault path and
  `C:\Users\graemel\Projects\PartnerTool` — update both if the new machine uses different paths.
- `C:\PCI\…` (logs/tools) is created and ACL-hardened by the app itself on first run — nothing to do.
- Defender/ASR may block the freshly built exe (new hash, zero prevalence) — see the confirmed
  ASR caveat in [[Build & Release]].

## What transfers vs. what rebuilds

| Thing | Transfers in the vault backup? |
|---|---|
| All project knowledge, conventions, lessons, changelog | ✔ yes |
| Full source code + logo assets | ✔ yes (Code snapshot) |
| Built exes / obj / bin | ✘ rebuild (one publish command) |
| Claude's memory index | ✘ rebuilds itself from this vault via step 1 |
| `C:\PCI` logs from the old machine | ✘ not in the vault — back up separately if the audit trail matters |
