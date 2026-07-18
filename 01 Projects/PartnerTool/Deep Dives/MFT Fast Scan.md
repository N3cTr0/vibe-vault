---
project: PartnerTool
tags: [partnertool, deep-dive, ntfs]
---

# MFT Fast Scan (Disk Usage ⚡)

WizTree-style whole-drive scan in seconds: open the raw volume read-only (`\\.\C:`), read the entire **$MFT** in one sequential pass, parse every file record, build the tree in memory. Implementation: `MftVolume.cs` (see [[_Code Index]]). Falls back to a parallel folder walk (`DiskUsageInfo`) for non-NTFS or any failure.

## How it works

1. `CreateFile(\\.\C:, GENERIC_READ)` — needs admin (we have it); read-only, so no data risk.
2. `FSCTL_GET_NTFS_VOLUME_DATA` → record size, cluster size, MFT start LCN.
3. Read record 0 ($MFT itself) → decode its **data runs** (the MFT can be fragmented) → bulk-read all records in ~1 MB blocks.
4. Per record: apply the **USA fixup**, read `$STANDARD_INFORMATION` (attributes + mtime), `$FILE_NAME` (name + parent ref; prefer Win32 over DOS namespace), `$DATA` (sizes), `$REPARSE_POINT` (cloud tag).
5. Build children lists from parent refs (root = record 5), compute recursive totals once; every drill-down is then instant.

## The accuracy saga — every bug we hit (validated against WizTree's *Allocated* column)

| Symptom | Root cause | Fix |
|---|---|---|
| VMs folder 18 GB instead of 127 GB | Huge fragmented files store `$DATA` in **$ATTRIBUTE_LIST extension records**; base record has no size | Attribute a `$DATA` size found in an extension record to its **base file** (base ref at record +0x20); only trust the `StartingVCN=0` header |
| OneDrive counted at full logical size | Cloud placeholders: MFT AllocatedSize == logical size | Zero files with the **IO_REPARSE_TAG_CLOUD*** family (`(tag & 0xFFFF0FFF) == 0x9000001A`) |
| Windows/Program Files dropped to ~8 GB | Over-zealous cloud zeroing via `$STANDARD_INFORMATION` attribute bits — that field is **not** a clean GetFileAttributes | Zero **only** by reparse tag, never by attribute bits |
| Windows/PF still ~0 for many files | **WOF/CompactOS** compression keeps real bytes in a *named* `WofCompressedData` stream; unnamed stream ≈ 0 | Sum **all** `$DATA` streams, not just the unnamed one |
| $BadClus showed 473.9 GB; $Extend 21 GB | Sparse system files declare AllocatedSize as the whole volume / journal max | For **sparse** streams, total only the real (non-sparse) data runs; `$BadClus` additionally zeroed by name (its stream isn't flagged sparse) |

**Remaining known difference vs WizTree:** hard-linked files (WinSxS) are counted once — our Windows folder reads a few GB lower, which is arguably *more* correct for on-disk usage.

## Sizing philosophy

Everything reports **size on disk (allocated)**, not logical size — so sparse files, compression, and cloud-only OneDrive content are honest. The folder-walk fallback matches via `GetCompressedFileSizeW` + skipping files with recall/offline attributes (metadata-only — never triggers a hydration/download).

## UI notes

WizTree-style GridView: Name · % of Parent bar · Size · Files · Modified · Attr (R/H/S/C). Hidden/system rows are dimmed and filtered by a "Show hidden items" checkbox (display-only — totals always include hidden). Single-click browses into folders; a `..` row goes up. Summary bar shows Total/Used/Free + scan mode + time.
