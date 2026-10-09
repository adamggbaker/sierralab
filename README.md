# Sierralab — Obsidian Vault + Windows Server 2025 (WINTEN)

Backup & Recovery workspace for:

- **Files** — Obsidian vault (synced to GitHub + local backup on E:)
- **Active Directory** — system-state backup (NTDS.DIT, SYSVOL, AD registry)
- **Group Policy** — GPO export + AD/GPO system-state backup

## Structure

| Path | Purpose |
|---|---|
| `Backup-Recovery.md` | This document (backup/recovery plan) |
| `README.md` | Repository overview |
- `Decommissioning-Plan-WINTEN.md` | Decommissioning plan (WINTEN DC + 3 VMs) |
| `.obsidian/` | Obsidian app settings (git-tracked) |

## Remotes

- **Primary (git replicate):** GitHub `adamggbaker/sierralab`
- **Local backup (E:):** `E:\Backups\Files\`, `E:\Backups\AD\`, `E:\Backups\GPO\`

## Tools

- **Obsidian** with [obsidian-git](https://github.com/Vinzent03/obsidian-git) plugin
- **Windows Server Backup** (WSB) on WINTEN
- **pywinrm / PowerShell remoting** from WSL2 for host-side commands
