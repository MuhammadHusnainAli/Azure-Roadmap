# Storage Accounts

## Why it matters

A storage account is the container for Azure Blob, Data Lake, Files, Queue and Table storage. Its type, performance tier and redundancy are set at creation and decide price, performance and what survives a zone or region failure. Several settings cannot be changed later without migrating data. Getting the account design right is the first storage decision in almost every Azure solution, AI included.

## Key concepts

### Account types

| Account kind | Services | Performance | Use |
|---|---|---|---|
| **Standard general-purpose v2** | Blob (incl. Data Lake), Files, Queue, Table | Standard (HDD-backed, SSD for some operations) | Default for almost everything |
| **Premium block blob** | Block and append blobs (incl. Data Lake) | Premium SSD, low latency | High transaction rates, small objects, interactive analytics, AI data loading |
| **Premium file shares** (FileStorage) | Azure Files | Premium SSD | Low-latency SMB and NFS shares |
| **Premium page blobs** | Page blobs | Premium SSD | Specialised; unmanaged VM disks (legacy) |

**General-purpose v1 and legacy Blob storage accounts are retired or retiring** (the retirement date for GPv1 was October 2026). Upgrade any you find to GPv2, which is an in-place change.

### Redundancy

| Option | Copies | Protects against | Read from secondary |
|---|---|---|---|
| **LRS** | 3 in one datacentre | Disk and rack failure | – |
| **ZRS** | 3 across availability zones in one region | Zone failure | – |
| **GRS** | LRS in primary + LRS in paired secondary region (async) | Regional disaster | Only after failover |
| **RA-GRS** | As GRS | As GRS | Yes, read-only endpoint `-secondary` |
| **GZRS** | ZRS in primary + LRS in secondary (async) | Zone and regional failure | Only after failover |
| **RA-GZRS** | As GZRS | As GZRS | Yes |

Durability ranges from at least eleven nines (LRS) to sixteen nines (geo options) per year. Availability SLAs differ by option and by whether you read from the secondary; check the current SLA page.

Key facts:

- Geo-replication is **asynchronous**. The **last sync time** tells you how far behind the secondary is; typically minutes, with no guaranteed RPO. Data written since the last sync can be lost in an unplanned failover.
- Not every service supports every option. Premium accounts support LRS and ZRS only. Some Azure Files tiers have their own restrictions.
- You can change redundancy later (for example LRS → ZRS) through conversion or migration, but it takes time and sometimes requires support or a copy.

### Failover

| Type | Trigger | Data loss | Use |
|---|---|---|---|
| **Customer-managed planned failover** | You, while primary is healthy | None | DR drills, moving primary |
| **Customer-managed (unplanned) failover** | You, when primary is unavailable | Possible (writes after last sync time) | Real regional outage |
| **Microsoft-managed failover** | Microsoft, only in extreme regional disasters | Possible | Do not plan around it |

After an unplanned failover the account becomes LRS in the new primary until you re-enable geo-redundancy. Design applications to tolerate the switch (endpoints stay the same; DNS changes).

### Access tiers (Blob)

| Tier | Storage cost | Access cost | Minimum retention | Latency |
|---|---|---|---|---|
| Hot | Highest | Lowest | – | Milliseconds |
| Cool | Lower | Higher | 30 days | Milliseconds |
| Cold | Lower still | Higher still | 90 days | Milliseconds |
| Archive | Lowest | Highest, plus rehydration | 180 days | Hours (rehydrate first) |

The account has a default access tier (hot or cool); individual blobs can override it. Lifecycle management moves data automatically (file 02). Azure has also introduced an automatic tiering option that moves objects between tiers based on access (check current status before relying on it).

### Endpoints and names

- Account names are globally unique, 3–24 lowercase letters and numbers.
- Endpoints: `https://<account>.blob.core.windows.net`, `.dfs.` (Data Lake), `.file.`, `.queue.`, `.table.`, `.web.` (static website).
- **Azure DNS zone endpoints** (preview in some regions) allow very large numbers of accounts per subscription with a different endpoint format; standard endpoints suit most cases.

### Scale limits

Accounts have limits on capacity, request rate and ingress and egress per region; defaults are high (petabytes, tens of thousands of requests per second) and some can be raised through support. A subscription can have a limited number of accounts per region (250 by default, higher on request). For very high scale, partition across accounts.

## How it works on Azure

### Settings to decide at creation

| Setting | Can it change later? |
|---|---|
| Name, region | No |
| Performance (Standard or Premium) | No (copy to a new account) |
| Account kind | GPv1 → GPv2 only |
| Hierarchical namespace (Data Lake) | One-way upgrade possible with constraints |
| Redundancy | Yes, through conversion or migration |
| Default access tier | Yes |
| Networking, security settings | Yes |
| Infrastructure encryption (double encryption) | No |

### Secure-by-default settings

```bash
az storage account create -g $RG -n $SA -l $LOC \
  --kind StorageV2 --sku Standard_ZRS --access-tier Hot \
  --min-tls-version TLS1_2 --https-only true \
  --allow-blob-public-access false --allow-shared-key-access false \
  --default-action Deny --public-network-access Disabled
```

These choices are covered in detail in file 06. Azure Policy can enforce them across subscriptions.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Redundancy for production | ZRS minimum where zones exist; GZRS (or RA-GZRS) when the data must survive a regional outage |
| RA- variants | Use when the application can read stale data from the secondary during an outage |
| Standard or Premium | Standard by default; Premium block blob when latency or transaction rates matter (small files, AI data loading, interactive analytics) |
| Number of accounts | Separate by workload, environment, data classification and lifecycle; not one account for the whole company, not one per file |
| Hierarchical namespace | Enable for analytics and data lake workloads (file 03); leave off for simple object storage needing features not yet supported with it |
| Region | Same region as compute to avoid egress and latency |
| Geo-redundancy vs backup | Replication copies mistakes too. You still need soft delete, versioning and backup (file 02) |

## Hands-on

```bash
RG=rg-storage-lab; LOC=swedencentral; SA=stlab$RANDOM
az group create -n $RG -l $LOC -o none

# 1. A GZRS account with secure defaults (keep public network access on for the lab)
az storage account create -g $RG -n $SA -l $LOC --kind StorageV2 --sku Standard_GZRS \
  --min-tls-version TLS1_2 --allow-blob-public-access false --allow-shared-key-access false -o none

# 2. Inspect properties
az storage account show -g $RG -n $SA \
  --query "{kind:kind, sku:sku.name, primary:primaryLocation, secondary:secondaryLocation, tier:accessTier, sharedKey:allowSharedKeyAccess}" -o table

# 3. Last sync time and failover readiness
az storage account show -g $RG -n $SA --expand geoReplicationStats \
  --query "geoReplicationStats.{status:status, lastSync:lastSyncTime, canFailover:canFailover}" -o table

# 4. Change redundancy (GZRS → RA-GZRS)
az storage account update -g $RG -n $SA --sku Standard_RAGZRS -o none

# 5. (Optional) Planned failover drill: no data loss, takes time
# az storage account failover -g $RG -n $SA --failover-type Planned

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** disable shared key access and public blob access, require TLS 1.2 or later, use private endpoints. Detailed in file 06.
- **Cost:** redundancy roughly scales cost: LRS < ZRS < GRS < GZRS. Geo options also bill for geo-replication data transfer.
- **Cost:** transactions and data retrieval can exceed capacity cost for chatty workloads; model both.
- **Reliability:** test failover in non-production with a planned failover, and document how applications behave.
- **Reliability:** geo-redundancy does not protect against deletion or corruption. Combine with soft delete, versioning, immutability and backups.

## Common mistakes

- LRS for production data that must survive a zone outage.
- Assuming GRS means "readable in the other region at any time" (that is RA-GRS).
- Treating geo-replication as backup.
- One storage account for every workload, making access control and cost tracking impossible.
- Leaving GPv1 accounts in place past retirement.
- Creating Standard accounts for workloads that need Premium latency, then having to migrate.

## Check yourself

1. What is the difference between ZRS and GZRS?
2. What does RA- add to GRS or GZRS?
3. Why can an unplanned failover lose data, and how do you estimate how much?
4. Which settings cannot be changed after account creation?
5. When would you choose a premium block blob account?
6. Why is geo-redundant storage not a backup?
7. What are the minimum retention periods for cool, cold and archive?

<details>
<summary>Answers</summary>

1. ZRS keeps three copies across zones in one region. GZRS adds asynchronous replication to a secondary region (as LRS there).
2. A read-only endpoint in the secondary region available at all times, not only after failover.
3. Replication is asynchronous; writes after the last sync time may not have reached the secondary. Check the last sync time property.
4. Name, region, performance tier and infrastructure encryption; the account kind can only be upgraded from GPv1 to GPv2.
5. When low latency and high transaction rates matter, such as many small objects, interactive analytics or feeding GPUs.
6. It faithfully replicates deletions, overwrites and corruption. Backups and versioning provide recovery to an earlier point.
7. Cool 30 days, cold 90 days, archive 180 days.

</details>

## References

- [Storage account overview](https://learn.microsoft.com/azure/storage/common/storage-account-overview)
- [Azure Storage redundancy](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
- [Disaster recovery and storage account failover](https://learn.microsoft.com/azure/storage/common/storage-disaster-recovery-guidance)
- [Change redundancy configuration](https://learn.microsoft.com/azure/storage/common/redundancy-migration)
- [Access tiers for blob data](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview)
- [Scalability targets for standard storage accounts](https://learn.microsoft.com/azure/storage/common/scalability-targets-standard-account)
- [Upgrade to a general-purpose v2 account](https://learn.microsoft.com/azure/storage/common/storage-account-upgrade)
