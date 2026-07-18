---
project: PartnerTool
tags: [partnertool, feature]
---

# Disk Usage (page)

TreeSize/WizTree-style space explorer. Full mechanics + accuracy history: [[MFT Fast Scan]].

- Drive picker (fixed NTFS disks only, no network drives) → ⚡ raw-MFT scan (~2–4 s whole drive) with automatic folder-walk fallback.
- WizTree-style columns: Name · % of Parent bar · Size · Files · Modified · Attr (R/H/S/C). **Sortable (0.19.0)** — click any header to sort, click again to reverse; the active one shows ▲/▼. Default is **% of Parent, descending** (`_sortCol=1`, matches WizTree). Sorting lives in `DiskUsagePage.SortItems`/`Header_Click`/`UpdateHeaders`, applied in `ApplyView`; the `..` row is pinned on top and never sorted, and every key falls back to bytes so ties stay sensible. Name/Attr sort as text (ordinal, case-insensitive), the rest numerically.
- Single-click browses into folders; `..` row goes up; Rescan re-reads the MFT; Open in Explorer.
- **Sizes are size-on-disk** (sparse/WOF/cloud-aware); OneDrive online-only files ≈ 0.
- Hidden/system items dimmed; "Show hidden items" checkbox is a display filter only — totals always include them.
- Summary bar: Total / Used% / Free% / scan mode + time.
