# Data Tier Resilience

## Why it matters

Compute can be recreated from code in minutes; data can't. The data tier determines your real RPO (how much you can lose) and usually your RTO (how long until you're consistent and writable again). Each Azure data service has different redundancy options, replication modes, failover mechanics and consistency trade-offs, and choosing the wrong one often only shows up during the first real outage. This file compares the options for **Storage**, **Azure SQL**, **Cosmos DB**, **PostgreSQL/MySQL**, and supporting data services (messaging, search, Key Vault), and shows how to combine them into a coherent data DR design. Database-specific detail is in [Topic 09 file 12](../09-databases/12-backup-ha-and-dr-for-databases.md).

## Key concepts

### Storage redundancy

| Option | Copies | Protects against | Failover |
|---|---|---|---|
| **LRS** | 3 in one datacentre | Disk/rack failure | None |
| **ZRS** | 3 across zones | Zone outage | Automatic (within region) |
| **GRS / RA-GRS** | LRS primary + LRS secondary region (async) | Regional outage | Customer-managed failover; RA- adds read access to secondary |
| **GZRS / RA-GZRS** | ZRS primary + LRS secondary region | Zone and regional outage | As GRS |

Geo-replication is asynchronous: check **Last Sync Time** to estimate data loss. Failover types: **customer-managed planned failover** (for DR testing, no data loss), **customer-managed unplanned failover** (outage; possible data loss), and Microsoft-managed failover (only for severe regional disasters, not per account; don't rely on it). Add **point-in-time restore**, versioning, soft delete and immutability policies for recoverability ([Topic 08 file 06](../08-storage/06-storage-security.md)).

### Databases summary

| Service | HA in region | Cross-region DR | Recoverability |
|---|---|---|---|
| **Azure SQL Database** | Zone-redundant configuration | Active geo-replication; **failover groups** with listener endpoints (async, RPO seconds) | PITR 1–35 days, LTR up to 10 years, geo-restore |
| **SQL Managed Instance** | Zone redundancy (Business Critical/GP where supported) | Failover groups | PITR, LTR |
| **Cosmos DB** | Availability zones per region | Multi-region accounts; single-write with service-managed or manual failover; **multi-region writes** with conflict resolution | Continuous backup (7 or 30 days PITR) or periodic |
| **PostgreSQL / MySQL flexible server** | Zone-redundant HA (synchronous standby) | Cross-region read replicas with promotion; geo-redundant backup with geo-restore | PITR |
| **Azure Managed Redis** | Zone redundancy | Active geo-replication (supported tiers) | Persistence (RDB/AOF) for some tiers |

### Supporting data services

| Service | Regional resilience | Cross-region |
|---|---|---|
| **Service Bus** | Zones (Premium) | **Geo-replication** (replicates metadata and data) or geo-disaster recovery (metadata only) |
| **Event Hubs** | Zones | **Geo-replication** (data and metadata) or metadata geo-DR |
| **Azure AI Search** | Replicas across zones (2+ replicas for read SLA, 3+ for read-write) | No built-in geo-replication; deploy multiple services and index to both |
| **Key Vault** | Zone-redundant in supported regions | Contents replicated to paired region with read-only failover; for nonpaired, plan your own (backup/restore or multiple vaults) |
| **ACR** | Zone redundancy (Premium) | Geo-replication (Premium) |

### Consistency and RPO

| Mode | RPO | Latency |
|---|---|---|
| Synchronous (zone-redundant, sync standby) | 0 | Small write penalty |
| Asynchronous geo-replication | Seconds to minutes | None on writes |
| Backup-based (geo-restore) | Up to backup interval (often ~1 h) | None |

## How it works on Azure

SQL failover group (Bicep fragment):

```bicep
resource fog 'Microsoft.Sql/servers/failoverGroups@2023-08-01-preview' = {
  parent: sqlPrimary
  name: 'fog-orders'
  properties: {
    readWriteEndpoint: { failoverPolicy: 'Manual' }       // customer-managed, recommended
    readOnlyEndpoint: { failoverPolicy: 'Disabled' }
    partnerServers: [ { id: sqlSecondary.id } ]
    databases: [ ordersDb.id ]
  }
}
```

Apps connect to `fog-orders.database.windows.net`, which follows the primary.

Cosmos DB multi-region with zone redundancy and continuous backup (CLI):

```bash
az cosmosdb create -g rg-data -n cosmos-orders \
  --locations regionName=westeurope failoverPriority=0 isZoneRedundant=true \
  --locations regionName=northeurope failoverPriority=1 isZoneRedundant=true \
  --enable-automatic-failover true --backup-policy-type Continuous --continuous-tier Continuous30Days
```

Storage DR test (planned failover, no data loss):

```bash
az storage account show -n storderprod --expand geoReplicationStats --query geoReplicationStats.lastSyncTime
az storage account failover -n storderprod --failover-type Planned
```

AI Search across regions: deploy a search service in each region, run indexers or push pipelines to both, and route queries via Front Door or app logic (file 06).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Start with RPO** | RPO 0 requires synchronous replication (zone-level); cross-region RPO 0 is rarely practical except with multi-write designs and trade-offs. |
| **Customer-managed failover** | Prefer explicit, tested runbooks over waiting for Microsoft-managed failover. |
| **Multi-write** | Only when you need multi-region writes and can handle conflicts and cost. |
| **Read replicas** | Use secondary reads for reporting and global read latency; reduces primary load. |
| **Backups always** | Replication doesn't protect against logical corruption; keep PITR and immutable backups. |
| **Consistent failover** | Fail over data, compute, secrets, identities and routing together via recovery plans. |

## Hands-on

1. Build GZRS storage, check last sync time, and run a planned failover test.
2. Create a SQL failover group with manual policy; perform a planned failover and measure application impact.
3. Create a two-region Cosmos DB account with continuous backup; restore a container to a point in time.
4. Configure Service Bus geo-replication and test promotion.
5. Deploy AI Search in two regions with dual indexing.

## Security, cost and reliability notes

- **Secondary security**: private endpoints, firewall rules, CMK and RBAC must exist in the secondary region.
- **Cost**: geo-replication roughly doubles data storage and adds egress; multi-write increases RU costs.
- **Data residency**: confirm secondary region is allowed.
- **Failback**: plan resync and direction reversal.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming GRS means automatic DR | Customer-managed failover runbooks |
| Automatic failover policies with unclear data-loss acceptance | Decide policy with business owners |
| Forgetting search indexes, queues and secrets | Include all data services |
| No PITR because "it's replicated" | Backups for corruption |
| Never testing database failover | Planned failover drills |

## Check yourself

1. Compare LRS, ZRS, GRS and GZRS.
2. What failover types does Storage support, and which should you rely on?
3. How do SQL failover groups help applications fail over?
4. What are Cosmos DB's multi-region options?
5. How do you make Azure AI Search resilient across regions?
6. Why keep backups when you have geo-replication?

<details>
<summary>Answers</summary>

1. LRS: three copies in one datacentre; ZRS: across zones; GRS: LRS + async copy in secondary region; GZRS: ZRS + async secondary.
2. Customer-managed planned, customer-managed unplanned, and Microsoft-managed; rely on customer-managed options.
3. They group databases and provide listener endpoints that point to the current primary.
4. Multi-region single-write with manual or service-managed failover, or multi-region writes with conflict resolution; zone redundancy per region; continuous backup.
5. Deploy services in multiple regions, index into each, and route queries; use replicas for in-region availability.
6. Replication copies corruption and deletions; backups allow point-in-time recovery.

</details>

## References

- [Azure Storage redundancy](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
- [Storage disaster recovery and failover](https://learn.microsoft.com/azure/storage/common/storage-disaster-recovery-guidance)
- [SQL Database failover groups](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db)
- [Cosmos DB high availability](https://learn.microsoft.com/azure/cosmos-db/high-availability)
- [PostgreSQL flexible server business continuity](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-business-continuity)
- [Service Bus geo-replication](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-geo-replication)
- [Reliability in Azure AI Search](https://learn.microsoft.com/azure/search/search-reliability)
