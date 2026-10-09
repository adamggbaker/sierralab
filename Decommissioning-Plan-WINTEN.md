# Decommissioning Plan — WINTEN (Windows Server 2025)

**Owner:** Administrator (Hyper-V / Windows Server admin)
**Host:** WINTEN @ 172.16.11.2 (domain controller, `Vlan11.local`)
**Date drafted:** 2026-10-08
**Prerequisite:** AD system-state snapshot + GPO export already captured (see `Backup-Recovery.md`)

---

## 1. Objective

Decommission WINTEN as a domain controller and remove its associated resources:
- Host files and profile on the secondary drive (`E:`)
- The `node1`, `node2`, `node3` Hyper-V VMs
- Active Directory (demote the DC)
- Group Policy objects and AD-backed GPO data
- All remaining remanence on `E:` and the primary drive

## 2. Scope & Inventory

| Item | Status | Location |
|---|---|---|
| AD DC | Active — to demote | `winten.Vlan11.local` |
| Domain: Vlan11.local | Active | 172.16.11.0/24 |
| Backup (AD snapshot) | Completed | `E:\partAbackups\AD\` |
| GPO export | Pending (step 3 of main task) | — |
| Files vault backup | Completed | `E:\partAbackups\Files\` |
| node1 VM | To be deleted | Hyper-V + `E:\new servers\node1` |
| node2 VM | To be deleted | Hyper-V + `E:\new servers\node2` |
| node3 VM | To be deleted | Hyper-V + `E:\new servers\node3` |
| `E:\partAbackups` | Kept (backup artifact) | `E:` |
| `E:\part` | To review | `E:` |
| `E:\new servers` | To purge | `E:` |

## 3. Pre-Decommissioning Checks

1. Verify current time / log the plan start.
2. Snapshot the VMs to be deleted (all 3) before any deletion.
3. Confirm no pending jobs on the host.
4. Confirm the AD site/domain still has a functioning DC to take over (if any exists).
5. State: **WINTEN is the sole DC if no other DC exists** — if so, build a temporary standby DC first.

## 4. Step 1 — Files Vault + Secondary Drive

1. Confirm the file backup is verified (`E:\partAbackups\Files\`) — done.
2. Backup remaining `E:\` items to `E:\partAbackups\Files\` if needed.
3. Remove the temporary `E:\partAbackups\ADSnapshot` mount directory (if present).
4. Review and delete `E:\part` (if it holds no needed data).

## 5. Step 2 — Group Policy Export

1. Install the Group Policy Management console if not present: `Install-WindowsFeature GPMC -IncludeManagementTools`
2. Run Export-GPO for every GPO:
   ```powershell
   $dest = "E:\Backups\GPO\"
   Get-GPO -All | ForEach-Object { Export-GPO -Name $_.DisplayName -Path $dest -Confirm:$false }
   ```
3. Verify the export (count = number of GPOs).
4. Record this export in the tracking note (the tracking note is currently `⏳ Pending` for this step).

## 6. Step 3 — Demote the Domain Controller

**WARNING:** This removes WINTEN from the domain.

1. Confirm a rollback plan (restore AD from `E:\partAbackups\AD\` snapshot if needed).
2. Run `uninstall-addsdevice` / `Uninstall-ADDSDomainController` (Windows Server 2025):
   ```powershell
   Uninstall-ADDSDomainController -Confirm:$false
   ```
   or, if using PowerShell: `uninstall-addsdevice`
3. After the DC promotion completes, verify:
   - `Get-ADDomain` no longer returns WINTEN as a domain controller.
   - `nltest /dsgetdc:Vlan11.local` no longer resolves to WINTEN.
   - DC's AD services (`NTDS` / `DNS`) stopped and disabled.
4. Demote cleanup:
   - Delete the domain-orchestrated computer account on a working member if needed.
   - Remove `WINTEN` from any remaining domain groups (Backup Operators, Enterprise Admins, etc.) per least privilege.

## 7. Step 4 — Hyper-V VMs (node1, node2, node3)

1. Shut down each VM cleanly:
   ```powershell
   Stop-VM -Name node1 -Force
   Stop-VM -Name node2 -Force
   Stop-VM -Name node3 -Force
   ```
2. Remove each VM:
   ```powershell
   Remove-VM -Name node1 -Force
   Remove-VM -Name node2 -Force
   Remove-VM -Name node3 -Force
   ```
3. Delete the VM files on the host:
   ```powershell
   Remove-Item -Path 'E:\new servers\node1' -Recurse -Force
   Remove-Item -Path 'E:\new servers\node2' -Recurse -Force
   Remove-Item -Path 'E:\new servers\node3' -Recurse -Force
   ```
4. Verify removal:
   ```powershell
   Get-VM | Where-Object {$_.Name -match 'node[123]'}
   ```
   Expect zero results.
5. Consider exporting the VM snapshots to `E:\partAbackups\Snapshots\` first if they must be retained for any period.

## 8. Step 5 — AD Containers & GPO Cleanup

Once the DC is demoted and no longer authoritative:

1. Clean up the orphaned AD `Group Policy` containers and `SYSVOL` replicas:
   ```powershell
   # Optional: remove stale Group Containers
   Get-ADOrganizationalUnit -Filter {Name -like "CN=Group Policy..."} | Remove-ADOrganizationalUnit
   ```
   (Only run after confirming no policy is still needed.)
2. Remove the GPO backup folder content:
   ```powershell
   Remove-Item -Path 'E:\Backups\GPO\' -Recurse -Force
   ```
3. Remove the tracking note entry once this step is complete.

## 9. Step 6 — Residue & Final Verification

1. Run a full surface review of `E:` and the primary drive for leftover items:
   - `E:\partAbackups` — KEEP (backup artifact)
   - `E:\part` — delete if confirmed
   - `E:\new servers` — should be empty or absent
   - `E:\Backup` — confirm content / delete if unused
2. Verify no domain controller remnants remain on the network:
   ```powershell
   Get-ADDomainController -Filter * | Where-Object {$_.Name -notlike 'winten*'}
   ```
   Expect no WINTEN results.
3. Run the AD demotion test:
   ```powershell
   nltest /dsgetdc:Vlan11.local
   ```
4. Confirm the host is back in a standalone (workgroup) state if that's the intended end state.

## 10. Rollback / Restoration

If the decommissioning is reversed or AD must be restored:

1. Re-promote WINTEN as a DC:
   ```powershell
   Install-ADDSDomainController -DomainName Vlan11.local -InstallDNS -Force
   ```
2. Restore the AD database from the snapshot at `E:\partAbackups\AD\` (ntds.dit + support files) via `ntdsutil`.
3. Re-import the GPO exports from `E:\Backups\GPO\` (once Step 2 is completed).
4. Re-replicate the file vault from `E:\partAbackups\Files\`.

## 11. Tracking Note Update

Update `Backup-Tracking-2026-10-07.md` with:
- AD system-state backup → FAILED (documented in Section 2)
- GP export → completed (record pass/fail + count + date)
- Decommissioning plan → completed after execution, with each step's result
- No remaining tasks

---

## 12. Immediate Action Items

1. ✅ AD snapshot backup captured to `E:\partAbackups\AD\` (40 MB ntds.dit)
2. ⏳ Group Policy export to `E:\Backups\GPO\` (awaiting GPMC)
3. 🔄 Demote WINTEN as DC (after GP export + verification)
4. 🔄 Delete node1/node2/node3 VMs + `E:\new servers` content
5. 🔄 Final cleanup: `E:\part`, residue verification, tracking note update
