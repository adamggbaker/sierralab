# Backup & Recovery Plan — WINTEN (Windows Server 2025)

**Owner:** Administrator (Hyper-V / Windows Server admin)
**In scope:** Files (Obsidian vault) · Active Directory · Group Policy
**Date:** 2026-10-07
**Companion:** README.md · Security/BCP-of-this-plan under separate management (physical / other)

---

## 1. Environment

| Item | Detail |
|---|---|
| Admin workstation | WSL2 Ubuntu 26.04.1 LTS, user `administrator` (passwordless sudo) |
| Management host | Windows Server 2025 "WINTEN" @ 172.16.11.2 (subnet 172.16.11.0/24) |
| Active Directory | AD on WINTEN (domain controller) |
| Group Policy | GPOs managed via GPMC on WINTEN |
| Vault | Obsidian "Sierralab" @ `C:\Users\Administrator\Desktop\obsidian\Sierralab` |
| Remote access | WSL → WINTEN: WinRM **5985 Open**; SSH **22 Blocked**. Hyper-V PowerShell cmdlets **cannot** run in WSL2 → must execute on the host via WinRM (`pywinrm` / PowerShell remoting) |
| Secondary backup target | E: (90 GB, 69 GB free) |

## 2. Backup Policy

| Rule | Principle | Value |
|---|---|---|
| 3-2-1 | 3 copies, 2 media types, 1 offsite | Vault git mirror (GitHub) + E: (local) + 1 offsite/monthly |
| Frequency | Full weekly, incremental/system state daily | Full: Sun 02:00, Daily: 02:00 (system state only) |
| Retention | 4 weekly + 12 monthly + 1 yearly | GPO backups: 12 monthly |
| Validation | Test restores monthly | AD: restore test once per quarter on a separate DC/VM |
| Encryption | Data at rest | BitLocker for OS/boot volumes; EFs for VSS/catalog on E: |
| Credentials | Least privilege | Dedicated `BackupOperators` + `DOMAIN\BackupAdmin` account, stored in vault vault-mgr (not plaintext) |
| Alerts | Notify on failure | Task Scheduler + Event Viewer (Event ID on the host → e-mail/Slack) |

## 3. Backup 1 — Files (Obsidian vault + host config)

**Primary target:** E:

**Method A — obsidian-git (already configured):**
- The vault is already mirrored to GitHub `adamggbaker/sierralab` via the `obsidian-git` plugin.
- This is the **replication/copy** layer; the **local copy** must still land on E:.
- Add `.gitignore` rule to keep large/unwanted attachments from bloating the repo.

**Method B — Windows Server Backup / E: (the "all backups to secondary hard drive" rule):**
- Backup the vault directory + host `Administrator` profile to `E:\Backups\Files\<date>\` using Windows Server Backup file-level policy, or a robocopy scripted job.
- Robocopy job (scheduled daily, run as `administrator`):
  ```bat
  robocopy "C:\Users\Administrator\Desktop\obsidian\Sierralab" "E:\Backups\Files\%date:~-4%%date:~4,2%%date:~7,2%" /E /COPY:DAT /R:3 /W:5 /NP /XO /MIR
  ```
- Retention: move oldest daily folder to `E:\Backups\Files\Archive\` weekly.
- Do **not** back up `AppData\Local\Temp`, browser caches, or `node_modules`.

## 4. Backup 2 — Active Directory (WINTEN)

**Scope captured:** `NTDS.DIT`, `SYSVOL`, AD system registry, boot files (system state).

**Method — Windows Server Backup (system state + all-critical), run on the host:**

```powershell
# Install the feature once (if not already)
Add-WindowsFeature Windows-Server-Backup -IncludeManagementTools

# One-time full backup to E: (VSS full; clears transaction logs)
wbadmin start backup -backupTarget:E: -allCritical -systemState -include:C: -vssFull -quiet

# Daily incremental/system state (VSS copy; does not clear logs)
wbadmin start backup -backupTarget:E: -allCritical -systemState -include:C: -vssCopyBackup -quiet
```

**Alternative — NTDS snap (dedicated DC backup):**
```powershell
ntdsutil "activate instance ntds" "snapshot" "create" "script" "quit" | cmd /c 'rep'
```
(After the snap is created, copy the snapshot folder to `E:\Backups\AD\snapshots\`.)

**VSS setting on the AD volumes** (recommended):
```powershell
vssadmin list providers
vssadmin list shadows
Set-Volume -DriveLetter C -MountTimeoutSeconds 300
```

**Retention:** 4 daily copies + 12 weekly + 12 monthly (WSB retention policy).

**Restore references (host):**
- Full/Bare-metal restore: `wbadmin start recovery -backupTarget:E: -version:<date> -items:C:` → boot from the Windows Server Backup recovery environment.
- AD restore: use `ntdsutil` / `wbadmin` to restore `NTDS.DIT`; promote or demote as required after restore.
- Test AD restore at least quarterly on a separate domain controller / VM.

## 5. Backup 3 — Group Policy (GPO)

**Scope captured:** GPO definitions (system volume `SYSVOL\Policies\`), the `Group Policy` AD containers, and the Group Policy client/service registry.

**Method — GPMC Backup-GPO, to E::**

```powershell
# Install the Group Policy Management console once
Install-WindowsFeature GPMC -IncludeManagementTools

# Backup ALL GPOs to E (use a UNC path to the E: share, or a local folder)
$dest = "E:\Backups\GPO\"
$GPOs = Get-GPO -All
$GPOs | ForEach-Object { Backup-GPO -All -Name $_.DisplayName -Path $dest -Confirm:$false }

# Verify
gpmc /report "Backup-GPO Status" | Out-File "$env:USERPROFILE\Desktop\GPOBackupVerify.txt"
```

**Retention:** 12 monthly snapshots per GPO; delete oldest beyond retention.

> Notes:
> - GPOs are stored in AD and also replicated to `SYSVOL`. Backing up GPOs via `Backup-GPO` plus the system-state AD backup above is the complete set.
> - A `gpresult /h` report should also be captured per GPO for audit.

## 6. Host-level supporting backups (recommended, outside the requested scope)

- **Hyper-V VMs** (node1/node2/node3 on `E:\new servers\`): export VMs to `E:\Backups\HyperV\` + snapshot `VHDX`s; weekly export snapshot + daily export.
- **DNS / DHCP** (if relayed on WINTEN): system-state backup already covers them via AD backup.
- **E: RECYCLE.BIN / System Volume Information**: leave alone (do not copy while Windows is running).

## 7. Restore & DR Runbooks

| Scenario | Restore method |
|---|---|
| Single file / note lost | `git checkout <commit> -- <file>` or obsidian-git plugin history |
| Vault metadata corrupted | Windows Server Backup file restore to E: |
| AD corruption / ntds.dit failure | AD system-state restore via `wbadmin`; out-of-order restore sequence if needed |
| GPO misconfigured | `Restore-GPO` from last known good snapshot on the backup path |
| Server offline | Bare-metal restore from WSB recovery environment |

## 8. Monitoring, Alerting & Verification

- Task Scheduler job + host task history (`schtasks /query /v`) for *every* scheduled backup, daily.
- Event Viewer: `Applications and Services Logs\Windows Server Backup` → daily success/fail; alert on failure.
- Check `E:` free space weekly; alert when < 20%.
- Verify MD5/SHA256 of a known test file after each backup run.
- **Quarterly:** full AD restore test; **monthly:** file + GPO verify/restore test.

## 9. Change & Escalation

- Any change to the vault structure, GPO, or AD schema → create a backup checkpoint *before* the change.
- Discontinued backup job → re-run immediately + re-baseline retention.
- Escalation: if a daily backup fails across 2 consecutive runs, raise a helpdesk ticket and notify on-call.

---

## 10. Immediate Action Items (next steps — requires host credentials)

1. Create a privileged `BackupAdmin` account + add to `BackupOperators`.
2. Install / confirm **Windows Server Backup** feature on WINTEN.
3. Create `E:\Backups\Files\`, `E:\Backups\AD\`, `E:\Backups\GPO\` with correct security ACLs.
4. Generate this plan into the vault (`Backup-Recovery.md`) and commit+push to GitHub.
5. Register the scheduled tasks on WINTEN (one-time `schtasks /create`).
6. Perform the first full backup and verify the restore on a test VM.
