---
project: Claude Dev VM
tags: [claude-dev-vm, moc]
---

# Claude Dev VM — Project Hub

> A dedicated Windows 11 virtual machine on the desktop PC (VMware Workstation Pro) that Claude Code gets **full autonomous access to** for development work. **As of 2026-07-18 this VM is the *sole* dev environment** — the host is no longer used for coding (it only runs the hypervisor). Isolation means Claude can run without permission prompts (`--dangerously-skip-permissions`), test elevated / system-changing actions (e.g. [[PartnerTool]] repair features), and any damage is a snapshot revert away. **This note is the reference point for everything about the VM.**

- **Status (2026-07-18):** VM fully provisioned — VMware Tools, dev stack, and Claude Code installed; `clean-dev-setup` snapshot taken; PartnerTool cloned to **`C:\Projects\PartnerTool`** (git, remote `N3cTr0/PartnerTool`, `main`). **Now the sole dev environment** — first autonomous session (2026-07-18) completed.
- **Host:** desktop PC, Windows 11 Pro, VMware Workstation Pro (free since Nov 2024 — no license key, downloaded via free Broadcom account)
- **Guest:** Windows 11 x64

## Host hardware

| Component      | Spec                                                           |
| -------------- | -------------------------------------------------------------- |
| CPU            | AMD Ryzen 7 3700X — 8 cores / 16 threads, 3.6 GHz base         |
| RAM            | 32 GB                                                          |
| Storage        | C: NVMe SSD (OS) · second NVMe SSD · E: SATA SSD · D: SATA HDD |
| Virtualization | Enabled in BIOS (confirmed via Task Manager)                   |

VM takes half the host: 8 of 16 logical CPUs, 16 of 32 GB RAM (~8–9 GB host headroom observed during install — drop the VM to 12 GB if the host ever feels memory-starved).

**VM files live on C: (NVMe, also the host OS drive).** Watch free space: the dynamic disk will grow to 60–100 GB+ and each kept snapshot adds more — keep C: comfortably above ~150 GB free, or relocate the VM to the second NVMe if it gets tight.

## VM specification (as built)

| Setting | Value |
|---|---|
| Disk | 200 GB, dynamically allocated |
| RAM | 16 GB |
| CPU | 4 cores × 2 threads = 8 logical processors |
| Firmware | UEFI + Secure Boot + virtual TPM 2.0 (auto-configured by Workstation's Windows 11 wizard; encryption = "only files needed for TPM") |

Sizing rationale: Windows 11 grows to ~40 GB after updates, Visual Studio adds 10–30 GB, plus SDKs/repos/build output — 200 GB dynamic costs nothing until used. Claude Code itself is tiny (~500 MB, fine in 4–8 GB RAM); the toolchain drives the spec.

## Setup checklist

- [x] Enable VT-x in BIOS / confirm virtualization enabled
- [x] Install VMware Workstation Pro (Broadcom download)
- [x] Download Windows 11 ISO (Microsoft official)
- [x] Create VM — Win11 x64 guest type, specs above
- [x] Windows 11 install **with local account** — used the Pro "domain join" shortcut during OOBE, which creates a local-only account (no registry tricks needed on Pro)
- [x] Install VMware Tools (graphics, clipboard, shared folders)
- [x] Windows Update until clean
- [x] Install dev stack: Git, .NET 10 SDK, WiX **v5 pinned** (`dotnet tool install --global wix --version 5.0.2` — v6+ carries the Open Source Maintenance Fee for commercial use; v5 matches the host and builds the v4-schema Product.wxs fine)
- [x] Install Claude Code — native installer: `irm https://claude.ai/install.ps1 | iex`, signed in
- [x] **Snapshot: `clean-dev-setup`** — the permanent reset button
- [x] Git sync for PartnerTool (clone into VM)
	- [x] Host: Git 2.55 installed (winget), identity configured
	- [x] Host: PartnerTool put under git — it wasn't a repo before! Initial commit `b08d8ab`, 128 files; `.gitignore` excludes bin/obj/dist/release/.vs and installer MSI/zip artifacts
	- [x] Host: GitHub CLI installed
	- [x] `gh auth login` (user), private GitHub repo created and pushed: https://github.com/N3cTr0/PartnerTool
	- [x] VM: cloned to `C:\Projects\PartnerTool` (remote `N3cTr0/PartnerTool` over HTTPS; commit `b08d8ab`, working tree clean)
- [x] First autonomous test run (2026-07-18)

## Working model

- **Code sync via git**, not shared folders — the VM is where working changes live; Claude works on branches and pushes them, review/merge on **GitHub** (the host is retired for dev). Keeps history clean and auditable.
- **VM GitHub auth = fine-grained PAT**, not full account sign-in: github.com → Settings → Developer settings → Fine-grained tokens; Contents read/write on **only** `PartnerTool` + `vibe-vault` (the [[Restore From Snapshot|vault backup]]). The autonomous VM can't touch anything else in the account, and revoking is one click. **Current PAT expires 2026-08-17 — rotate before then or all pushes stop.**
- **Snapshot before autonomous runs**; revert if anything goes sideways (~5 seconds).
- Inside the VM, launch `claude --dangerously-skip-permissions` from an **elevated** PowerShell (Run as administrator) — elevation gives the system/registry/service/`C:\PCI` access PartnerTool exercises, and the flag suits this isolated dedicated-VM. The `clean-dev-setup` snapshot + repo-scoped PAT are the safety net, not the prompts.
- Windows in the VM runs unactivated (watermark + personalization limits only); assign a license if it becomes a permanent fixture.

## Decisions & gotchas

- 2026-07-18 — went with 200 GB dynamic / 16 GB RAM / 8 logical CPUs; host keeps enough RAM for itself.
- 2026-07-18 — **first install scrapped:** OOBE linked a Microsoft account. On the redo, the Windows 11 **Pro "domain join" shortcut** during OOBE gave a local-only account cleanly — remember this for future installs. (Fallbacks if ever needed: BypassNRO registry trick with network disconnected; `ms-cxh:localonly` was patched out in newest 25H2 builds.)
- Workstation's Win11 wizard handles TPM/Secure Boot automatically — no host TPM involvement, no vmx hacks needed.
- If Hyper-V/WSL2 is active on the host, Workstation runs in a slightly slower compatibility mode — works fine, just expected.
- 2026-07-18 — **confirmed Claude can launch + drive PartnerTool's WPF UI.** In an interactive (RDP) session WPF renders fine; `tools\ui-drive.ps1` navigates pages by AutomationId and screenshots (verified System Info → Diagnostics → Network). **Claude Code must run elevated** — not only to launch the `requireAdministrator` app, but because Windows **UIPI** blocks a medium-integrity process from injecting input into the elevated window. The Claude Code shortcut is set to **always Run as administrator** (shortcut Properties → Advanced), so every session comes up elevated automatically — silent under "Never notify", no right-click needed.
- 2026-07-18 — **UAC: keep it on, don't fully disable.** The Control Panel slider "Never notify" only sets `ConsentPromptBehaviorAdmin=0` (silent elevation) and leaves `EnableLUA=1` — that's enough to make "Run as administrator" promptless, which is all we need. Fully disabling UAC (`EnableLUA=0`, reboot) is a **trap**: it breaks UWP/Store-app launch, and PartnerTool depends on that (winget is MSIX; Store updates use `AppInstallManager`).
