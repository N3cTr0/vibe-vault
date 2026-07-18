---
project: PartnerTool
tags: [partnertool, feature]
---

# System Shortcuts (page)

One-click launchers for the consoles techs open constantly. Categories and the buttons inside each are **sorted alphabetically** (0.17.55):

- **Network & Remote:** Network Connections (ncpa.cpl), Remote Desktop (mstsc), Remote Desktop Settings (`ms-settings:remotedesktop`), Windows Firewall (wf.msc).
- **Printers:** classic *Devices & Printers* via `explorer shell:::{A8A91A66-3A7D-4424-8D24-04E180695C7A}` (not the Settings page), and **Print Management**, which self-installs the `Print.Management.Console` capability via DISM with a progress bar if missing.
- **Settings & Admin:** Add/Remove Programs, Control Panel, Environment Variables (rundll32), Group Policy, Registry Editor, System Properties.
- **System:** Computer Management, Device Manager, Disk Management, Event Viewer, Services, Task Manager.

Layout uses a `UniformGrid` (3 cols) per card so buttons auto-flow — easy to keep sorted. Launcher supports three tag schemes: plain exe/msc/`ms-settings:` URI, `rundll32://…`, `explorer://shell:::{GUID}`. All bare names are pinned to System32/Windows via `ProcessRunner.ResolveSystemTool` before ShellExecute (see [[Conventions & Security Model]]).
