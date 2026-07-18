---
project: PartnerTool
tags: [partnertool, moc]
---

# PartnerTool — Project Hub

> Progressive Computing's elevated Windows 11 support tool for techs: system info, diagnostics, repair actions, software install, disk usage, and more. WPF / .NET 10, distributed as a single self-contained exe. **This note is the reference point for everything PartnerTool.**

- **Repo:** `C:\Projects\PartnerTool` on the [[Claude Dev VM]] (the **sole dev environment** since 2026-07-18) — under **git**, pushed to **private GitHub `N3cTr0/PartnerTool`** (`main`; `git clone https://github.com/N3cTr0/PartnerTool.git`). GitHub is the primary source-of-truth for code; `bin/obj/dist/release/.vs` are git-ignored.
- **Current version:** 0.19.14 (pre-release `0.x` scheme — see [[Conventions & Security Model]])
- **Distribution:** single-file exe in `dist\` (see [[Build & Release]])
- **Disaster recovery:** primary = `git clone` from GitHub; offline fallback = [[Restore From Snapshot]] (the vault's `Code\` snapshot reconstructs the source — round-trip re-tested 18 Jul 2026 at v0.19.14, all files content-clean; auto-syncs on every commit)
- **New PC / fresh install:** [[New Machine Bootstrap]] — prerequisites + the one paragraph to paste into a fresh Claude session

## Core notes

- [[Overview]] — what it is, tech stack, how it's shipped
- [[Architecture]] — startup flow, pages, collectors, key classes
- [[Conventions & Security Model]] — the rules every change must follow
- [[Build & Release]] — dev builds, single-file publish, MSI, versioning
- [[Lessons Learned]] — bugs we hit and what they taught us
- [[Changelog]] — full version history (copy of `versions.md`)

## Features (one note per page)

- [[Health Check]] · [[System Info]] · [[Diagnostics]] · [[Disk Usage]] · [[Manage]] · [[Network]]
- [[Repair]] · [[Security]] · [[Software]] · [[System Shortcuts]] · [[Updates]]

## Deep dives

- [[MFT Fast Scan]] — the WizTree-style raw-NTFS scanner and its accuracy saga
- [[Tech Gate & Activity Log]] — destructive-action gating + the audit trail
- [[WPF Single-File Quirks]] — shutdown-telemetry crash, Wi-Fi location gating, binding gotchas
- [[Repair Internals]] — installer cleanup (Adobe), Office language removal, temp cleaner, WPAD fix

## Code

- [[_Code Index]] — snapshot of every source file (fenced code, searchable in Obsidian)
