# Backup & Decommissioning Tracking Note — 2026-10-07

## Current status
| Task | Status | Notes |
|------|--------|-------|
| Files backup → `E:\part a backups\Files\` | ✅ Completed & verified | 20 files + 8 folders (Backup-Recovery.md, README.md, Welcome.md, test.md, .obsidian/, etc.) |
| Active Directory system-state backup | ❌ FAILED | wbadmin returned "There is not enough free space on the backup storage location"; Wbengine.exe (PID 5900) was running, WindowsImageBackup\winten\Backup 2026-10-08 010731 existed but the job never completed. Does NOT block next steps. |
| Group Policy export (Export-GPO) | ⏳ Pending | Next step — run step-by-step below |
| Decommissioning plan | ⏳ After GP backup | Deletes VMs, files, demotes DC, verifies no remanence |

## What was verified on the host
- WinRM (port 5985) NTLM login with `administrator` / `Sierra0001` works on WINTEN (172.16.11.2)
- AD fully functional on WINTEN (NTDS.DIT present, Get-ADDomain works)
- E: is NTFS with ~68.7 GB free
- Files backup copied to E: and verified (20 files + 8 folders)

## AD backup failure detail
- Command: `wbadmin start backup -backupTarget:E: -allCritical -systemState -include:C: -vssFull -quiet`
- Error returned: "There is not enough free space on the backup storage location to back up the data"
- Confirmed: Wbengine.exe was running, WindowsImageBackup\winten\Backup 2026-10-08 010731 existed
- **Do NOT re-run the AD backup** — document the failure for the decommissioning plan.

## Note
- AD backup failed; it does not block the group policy backup.
- Group Policy backup will be run step-by-step (Export-GPO), documented at each step.
- Decommissioning plan generated after GP backup completes.
