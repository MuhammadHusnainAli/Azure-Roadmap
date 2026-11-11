# Azure Backup

## Why it matters

Replication protects against hardware and regional failures, but it faithfully copies mistakes: a dropped table, a corrupted file, an accidental delete or ransomware encryption reaches every replica within seconds. Backups let you go back in time. They're also the last line of defence in a cyberattack, which is why attackers target them first: they try to delete recovery points, disable soft delete or change retention before encrypting production. **Azure Backup** provides centrally managed, policy-driven backups for VMs, databases, files, blobs, disks, AKS and more, with **vaults** as security boundaries, **soft delete** enforced by default, **immutable vaults**, **multi-user authorisation** and **cross-region restore**.

## Key concepts

### Vaults

| Vault | Protects (examples) |
|---|---|
| **Recovery Services vault** | Azure VMs, SQL Server and SAP HANA in VMs, Azure Files, on-premises (MARS agent, MABS/DPM) |
| **Backup vault** | Azure Blobs, Azure Disks, Azure Database for PostgreSQL (flexible server long-term), AKS, ADLS vaulted backup and newer workloads |

Vaults are security boundaries: separate subscriptions/resource groups and RBAC from production, with their own storage redundancy (LRS, ZRS, GRS) and **Cross Region Restore** option for GRS vaults.

### Backup tiers

| Tier | Where data lives | Use |
|---|---|---|
| **Operational (snapshot) backup** | In the source account/subscription (for example blob operational backup, disk snapshots) | Fast, frequent restore; not isolated from the source |
| **Vaulted backup** | Copied into the vault | Isolation, long-term retention, ransomware protection |
| **Operational + vaulted** | Both | Fast restore plus durability |
| **Archive tier** | Long-term recovery points moved to cheaper storage | Compliance retention |

### Protection against deletion and tampering

| Feature | Behaviour |
|---|---|
| **Soft delete (secure by default)** | Enforced by default; deleted backup data kept 14 days at no extra cost; configurable **14–180 days** (charged beyond 14); soft-deleted vaults recoverable |
| **Immutable vault** | Blocks operations that could lose recovery points (for example reducing retention or deleting data before expiry); can be **locked** to make it irreversible; locked immutability increasingly backed by **WORM** storage. Doesn't apply to operational backups |
| **Multi-user authorisation (MUA)** | Critical operations (disable soft delete, reduce retention, disable MUA) require approval through a **Resource Guard** in a separate tenant/subscription controlled by a security admin |
| **Encryption** | At rest with platform keys or CMK |
| **Private endpoints** | Restrict vault network access |
| **Alerts and reports** | Built-in Azure Monitor alerts for job failures and security events; Backup reports |

### Management at scale

**Azure Business Continuity Center** (successor experience to Backup center) gives a unified view of protection across backup and Site Recovery, protectable-but-unprotected resources, jobs and alerts. **Azure Policy** can automatically configure backup for VMs with specific tags.

## How it works on Azure

Recovery Services vault with immutability, soft delete and VM backup via CLI:

```bash
az backup vault create -g rg-backup-prod -n rsv-contoso-prod -l westeurope
az backup vault backup-properties set -g rg-backup-prod -n rsv-contoso-prod \
  --backup-storage-redundancy GeoRedundant --cross-region-restore-flag true
az backup vault update -g rg-backup-prod -n rsv-contoso-prod --immutability-state Locked   # irreversible
az backup protection enable-for-vm -g rg-backup-prod -v rsv-contoso-prod \
  --vm /subscriptions/<sub>/resourceGroups/rg-app/providers/Microsoft.Compute/virtualMachines/vm-app01 \
  --policy-name EnhancedPolicy-Daily
```

Policy design example (enhanced policy for VMs): backups every 4 hours (enhanced policies support multiple daily backups), instant restore snapshots retained 7 days, daily recovery points 30 days, weekly 12 weeks, monthly 12 months, yearly 7 years (archive eligible).

Enable backup at scale with Azure Policy built-in "Configure backup on virtual machines with a given tag to a new recovery services vault with a default policy" or the existing-vault variant.

Restore options for VMs: create new VM, restore disks, replace existing disks, file-level recovery, cross-region restore to the secondary region (GRS vaults), and cross-subscription restore.

Database-native backups complement Azure Backup: Azure SQL automated backups with point-in-time restore (PITR) and long-term retention (LTR); Cosmos DB continuous backup; PostgreSQL/MySQL flexible server PITR (file 05).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Vault placement** | Separate subscription and RBAC from production; one vault per region/environment per policy domain. |
| **Redundancy** | GRS with Cross Region Restore for regional DR; ZRS for zone resilience within region; LRS only for non-critical. |
| **Immutability** | Enable for production; lock after validating retention settings (irreversible). |
| **MUA** | Resource Guard owned by the security team in a separate subscription/tenant. |
| **Retention** | Meet RPO and compliance; archive long-term points to save cost. |
| **Testing** | Scheduled restore tests; a backup you haven't restored is a hope, not a plan. |

## Hands-on

1. Create a Recovery Services vault with GRS and Cross Region Restore; back up a VM with an enhanced policy.
2. Enable soft delete retention of 30 days; delete backup data and recover it.
3. Enable immutability (unlocked) and try to reduce retention; then plan locking.
4. Configure MUA with a Resource Guard in another subscription; attempt to disable soft delete.
5. Perform file-level recovery and a cross-region restore test; record the time taken.

## Security, cost and reliability notes

- **Ransomware**: immutability + MUA + separate admin domains make backups survivable.
- **Least privilege**: Backup Operator vs Backup Contributor; restrict restore permissions too (restores can exfiltrate data).
- **Cost**: protected instance fee plus backup storage; archive tier and retention tuning reduce cost; soft delete beyond 14 days is charged.
- **Monitoring**: alert on failed jobs and security events (soft delete disabled, policy changes).

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating replication as backup | Point-in-time, isolated backups |
| Vault in the same subscription with the same admins as production | Separate subscription and RBAC, MUA |
| Never testing restores | Scheduled restore drills |
| LRS vault for critical workloads | GRS/ZRS as required |
| Locking immutability before validating policies | Test, then lock |

## Check yourself

1. Which workloads use Recovery Services vaults vs Backup vaults?
2. Compare operational and vaulted backup.
3. How does soft delete work by default, and how long can it be extended?
4. What does a locked immutable vault prevent?
5. What is multi-user authorisation?
6. Which restore options exist for Azure VMs?

<details>
<summary>Answers</summary>

1. Recovery Services: VMs, SQL/SAP HANA in VMs, Azure Files, on-premises (MARS/MABS). Backup vaults: blobs, disks, PostgreSQL, AKS, ADLS and newer workloads.
2. Operational backups stay in the source environment for fast restore; vaulted backups copy data into an isolated vault for durability and long-term retention.
3. Enforced by default with 14 days' retention at no extra cost; configurable up to 180 days.
4. Operations that could lose recovery points (reducing retention, early deletion, disabling immutability), irreversibly.
5. A Resource Guard requiring separate approval for critical backup operations.
6. New VM, restore disks, replace disks, file-level recovery, cross-region restore, cross-subscription restore.

</details>

## References

- [Azure Backup overview](https://learn.microsoft.com/azure/backup/backup-overview)
- [Secure by default with soft delete](https://learn.microsoft.com/azure/backup/backup-azure-enhanced-soft-delete-about)
- [Immutable vaults](https://learn.microsoft.com/azure/backup/backup-azure-immutable-vault-concept)
- [Multi-user authorisation](https://learn.microsoft.com/azure/backup/multi-user-authorization-concept)
- [Azure Business Continuity Center](https://learn.microsoft.com/azure/business-continuity-center/business-continuity-center-overview)
- [Restore Azure VMs](https://learn.microsoft.com/azure/backup/backup-azure-arm-restore-vms)
