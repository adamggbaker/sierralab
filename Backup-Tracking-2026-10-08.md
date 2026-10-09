# Backup & Decommissioning Tracking Note — 2026-10-08, updated

## Current status

| Task | Status | Notes |
|---|---|---|
| Files backup → E:\partAbackups\Files\ | ✅ Completed & verified | 20 files + 8 folders |
| Active Directory system-state backup | ❌ FAILED | wbadmin ran out of space; AD backed up via snapshot to E:\partAbackups\AD\ |
| Group Policy export (Export-GPO) | ⏳ Pending | Next step — run Export-GPO |
| Decommission all Hyper-V VMs (node1/node2/node3 + remaining) | ✅ Completed | 8 VMs deleted + E:\new servers emptied |
| Decommissioning plan (Decommissioning-Plan-WINTEN.md) | ✅ Removed | Placed in vault earlier |

## Verification

- **All 8 Hyper-V VMs deleted:** node1, node2, node3, AlpineClone1, AlpineClone2, AlpineClone3, AlpineVM, man
- **E:\new servers** — directory and contents removed (now empty)

## What's next

1. Group Policy export to E:\Backups\GPO\ (GPMC)
2. Demote WINTEN as DC
3. Final cleanup: tracking note, E:\part, residue verification
