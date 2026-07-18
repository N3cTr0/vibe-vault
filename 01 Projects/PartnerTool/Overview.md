---
project: PartnerTool
tags: [partnertool]
---

# Overview

**PartnerTool** ("Partner Support Tool") is Progressive Computing's internal Windows support app. A tech runs one exe on a client machine and gets a full diagnostic picture plus safe, logged repair actions — instead of juggling a dozen separate utilities.

## Tech stack

| Thing | Choice |
|---|---|
| UI | WPF, .NET 10 (`net10.0-windows10.0.17763.0`), Catppuccin Mocha dark theme |
| Elevation | `requireAdministrator` manifest — always runs elevated |
| Data | `System.Management` (WMI), registry, P/Invoke, bounded CLI tools via `ProcessRunner` |
| Sensors | LibreHardwareMonitorLib (temps; limited on HVCI machines — see [[WPF Single-File Quirks]]) |
| Distribution | Single self-contained exe (~66 MB) — see [[Build & Release]] |
| Installer | WiX MSI installing to `C:\PCI\PartnerTool` (optional; single exe is the norm) |

## What it does (page by page)

Startup captures one **SystemSnapshot** (all collectors in parallel behind a splash: "Analyzing System, Please Wait…"), then pages paint instantly from it. After the snapshot, `MainWindow.StartupAsync` also **pre-caches the slow pages in the background** while the tech reads System Info — all Manage sections, the Software installed-list + winget detection, and the Updates scans (each guarded to run once) — so those tabs are already populated when opened. See the feature notes: [[System Info]], [[Diagnostics]], [[Disk Usage]], [[Manage]], [[Network]], [[Repair]], [[Security]], [[Software]], [[System Shortcuts]], [[Updates]].

## Safety posture (short version)

- Destructive actions require the once-per-session **tech code** (date-derived) → [[Tech Gate & Activity Log]]
- Everything the tool does is written to `C:\PCI\Logs\PartnerTool_activity.log`
- Critical services/processes/security-software startup entries are hard-blocked from stop/kill/disable
- Full rules in [[Conventions & Security Model]]
