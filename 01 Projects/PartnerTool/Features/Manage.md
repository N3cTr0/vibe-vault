---
project: PartnerTool
tags: [partnertool, feature]
---

# Manage (page)

Sub-sections (0.17.66: Services pinned first, rest alphabetical): Services, Drivers, Environment Variables, Features, Hosts File, Scheduled Tasks, User Profiles. Each loads on first view and is cached; **all sections are pre-warmed in the background at startup** (`ManagePage.PreloadAsync` → `EnsureLoaded` per section, kicked off from `MainWindow.StartupAsync`) so the first jump into Manage and switching between any section is instant (0.17.56 — was Services + Tasks only in 0.17.55).

- **Services:** sortable like services.msc (Name/Status/Startup headers with ▲▼). Start is ungated; **Stop/Restart are tech-gated + confirmed**, and the critical blocklist (core Windows + firewall + Defender/EDR) is enforced at the UI *and* inside `ServicesInfo.Control` — see [[Conventions & Security Model]].
- **Hosts file:** edit + save (gated; overwrites the system hosts file).
- **User profiles:** lists with size/last-used/SID; **Delete** (gated) uses `Win32_UserProfile.Delete()` — the proper API that removes folder + ProfileList registry + hive. Loaded/special/current profiles can't be deleted.
- **Environment variables (0.19.4):** the read-only list now has an **Add / Update** row at the top — **Scope** picker first (left), then **Name**, then **Value**, each with a column label above it (0.19.7 — the scope choice comes first the way a tech thinks about it, and the two boxes were previously distinguishable only by tooltip). System (Machine) scope is **tech-gated + confirmed** (applies to all users); the list refreshes after a write. `SystemManagement.SetEnvVar` validates the name (no spaces/`=`/control chars) and uses `Environment.SetEnvironmentVariable`, which broadcasts WM_SETTINGCHANGE — new processes pick it up, already-open apps keep the old value.
- Drivers (signed/unsigned counts) and scheduled tasks are read-only lists.
