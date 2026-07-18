---
project: PartnerTool
tags: [partnertool, feature]
---

# Software (page)

Order: Install Software → Startup Programs → Installed Software. Loading (`EnsureLoadedAsync`) is **pre-cached from startup** (`MainWindow.StartupAsync`) so the list + winget "already installed" detection are ready when the tab is opened.

- **Install Software:** curated winget catalog by category (browsers, comms, VPN, MS tools, multimedia, pro tools, utilities). Already-installed apps detected two ways (registry name match instantly + `winget export` ID match, ~30 s) and shown ticked/locked. Installs are silent (`winget install --silent …`), each logged.
- **Startup Programs:** Run keys (HKLM/HKLM32/HKCU) + Startup folders, enabled state via the same `StartupApproved` binary blobs Task Manager uses. Loads instantly (before winget detection). **Security-software entries can't be disabled** (UI block + `SetEnabled` backstop) — see [[Conventions & Security Model]].
- **Installed Software:** searchable list from the shared `SoftwareInventory` (same source as the report). Search box shows a "Type to search installed software…" placeholder (a VisualBrush that clears on focus — the local-`Background` precedence bug that hid it is fixed, see [[WPF Single-File Quirks]] §2b). **Uninstall** is tech-gated; per-user (HKCU) uninstall strings are refused outright (LPE — user-writable command would run elevated).
