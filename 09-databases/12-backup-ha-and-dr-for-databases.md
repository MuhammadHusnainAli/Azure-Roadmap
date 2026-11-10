# Backup, HA and DR for Databases

## Why it matters

When a database is unavailable, the application is down; when data is lost, the business may be too. Azure databases include high availability, automated backups and geo-replication, but each option has a different RPO, RTO, cost and operational behaviour. Architects must translate business requirements into a design, and prove it works with tests. This file brings together the continuity features of the services in this topic.

## Key concepts

### Definitions

| Term | Meaning |
|---|---|
| **High availability (HA)** | Surviving node or zone failures within a region, usually automatically |
| **Disaster recovery (DR)** | Recovering from regional outages or major incidents, often with a decision to fail over |
| **Backup and restore** | Recovering data to an earlier point after deletion, corruption or attack |
| **RPO** (recovery point objective) | Maximum acceptable data loss, measured in time |
| **RTO** (recovery time objective) | Maximum acceptable time to restore service |

HA and DR protect against infrastructure failure; backups protect against **mistakes and malice**, which replication faithfully copies.

### Azure SQL Database

| Capability | RPO | RTO | Notes |
|---|---|---|---|
| Built-in HA (zone redundant) | 0 | Seconds | Automatic; enable zone redundancy |
| **Point-in-time restore (PITR)** | Minutes (log backups every 5–10 minutes) | Minutes to hours (size dependent; fast for Hyperscale) | 1–35 days retention; restores to a new database |
| **Long-term retention (LTR)** | Weekly/monthly/yearly | Hours | Up to 10 years |
| **Geo-restore** | Up to about 1 hour | Hours | From geo-replicated backups, when backup storage redundancy is geo |
| **Active geo-replication** | Seconds (async) | Seconds to minutes after manual failover | Up to 4 readable secondaries; per database |
| **Failover groups** | Seconds (async) | Minutes; listener endpoints redirect | Group of databases; customer-managed failover policy recommended |
| Deleted database restore | Last backup | Minutes to hours | Within retention |

**Backup storage redundancy** (LRS, ZRS, GRS, GZRS) is set per database; geo options are needed for geo-restore.

### Azure SQL Managed Instance

Same concepts: built-in HA (Business Critical with replicas, zone redundancy), PITR, LTR, geo-restore, **failover groups** (instance-level), and the **Managed Instance link** for DR to or from SQL Server.

### PostgreSQL and MySQL flexible server

| Capability | RPO | RTO |
|---|---|---|
| Zone-redundant HA | 0 | Typically 1–2 minutes |
| PITR (1–35 days) | Minutes | Minutes to hours |
| Geo-redundant backup restore | Up to about 1 hour | Hours |
| **Cross-region read replica promotion** | Seconds to minutes (async) | Minutes |
| Long-term backup (vaulted) | Per policy | Hours (check current options, such as Azure Backup integration) |

### Cosmos DB

| Capability | RPO | RTO |
|---|---|---|
| Availability zones in a region | 0 | 0 |
| Multi-region, single write region, service-managed failover | Depends on consistency; seconds to minutes (up to 0 with strong consistency) | Minutes |
| Multi-region writes | Near 0 (conflicts resolved) | Near 0 |
| **Continuous backup** (7 or 30 days) | Seconds (point-in-time) | Hours; restore to a new account |
| Periodic backup | Backup interval (configurable) | Hours; restore via support |

### Azure Managed Redis

Zone-redundant HA, persistence (RDB/AOF) for recoverability, **active geo-replication** for multi-region, and export/import for backups.

### Mapping requirements to designs

| Requirement | Design |
|---|---|
| RPO hours, RTO a day, low cost | Single region, zone-redundant HA, PITR, geo-redundant backups (geo-restore for DR) |
| RPO minutes, RTO an hour | Plus a cross-region readable secondary (geo-replication or read replica) with tested manual failover |
| RPO seconds, RTO minutes | Failover groups (SQL), promoted replicas with automation (PostgreSQL), service-managed failover (Cosmos DB) |
| RPO ~0, RTO ~0, global | Cosmos DB multi-region writes; or design application-level active-active |
| Ransomware and insider protection | Immutable or vaulted backups, LTR, separate subscription or tenant for backup copies, delete locks |

### Failover considerations

| Topic | Detail |
|---|---|
| **Customer-managed vs automatic failover** | Automatic (Microsoft-managed) failover only triggers in broad outages and can be slow to decide. Most designs use **customer-managed** failover with a runbook and clear decision criteria |
| **Connection endpoints** | Use failover group listeners, virtual endpoints or the Cosmos DB SDK's preferred regions so apps reconnect without config changes |
| **Application tier** | DR for the database is useless without the app, identity, secrets, network and DNS also available in the secondary region |
| **Data loss acceptance** | Forced failover can lose recent asynchronous writes; decide in advance who can authorise it |
| **Failback** | Plan how to return, including resync |
| **Testing** | Planned failover drills in production-like environments at least yearly; restore tests regularly |

## How it works on Azure

### Azure SQL failover group

```bash
# Secondary server in another region
az sql server create -g $RG -n $SRV2 -l $LOC2 --enable-ad-only-auth \
  --external-admin-principal-type Group --external-admin-name sql-admins --external-admin-sid $GROUP_ID -o none

az sql failover-group create -g $RG -s $SRV -n fg-orders --partner-server $SRV2 \
  --add-db $DB --failover-policy Manual -o none

# Apps connect to: fg-orders.database.windows.net (read-write) and fg-orders.secondary.database.windows.net (read-only)

# Planned failover drill (no data loss)
az sql failover-group set-primary -g $RG -s $SRV2 -n fg-orders
```

### Point-in-time restore

```bash
az sql db restore -g $RG -s $SRV -n $DB --dest-name ${DB}-restored \
  --time "2026-10-15T08:30:00Z"

az postgres flexible-server restore -g $RG -n pg-restored --source-server $PG \
  --restore-time "2026-10-15T08:30:00Z"
```

Restores create a new database or server. Plan how the application switches to it, or how you copy data back.

### Cosmos DB continuous backup restore

```bash
az cosmosdb restore -g $RG --target-database-account-name ${ACC}-restored \
  --account-name $ACC --restore-timestamp "2026-10-15T08:30:00Z" --location $LOC
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Zone redundancy | Default for production; the cost premium is small relative to the protection |
| Secondary region | Only when the business case (RPO and RTO) requires it; it can double cost |
| Readable secondaries | Use them for reporting to offset their cost |
| Failover policy | Customer-managed with runbooks and monitoring; automatic only where well understood |
| Backup retention | PITR to cover detection time for corruption (often 14–35 days); LTR per compliance |
| Backup isolation | For ransomware resilience, immutable or vaulted copies outside the reach of compromised admin accounts |
| Testing | Schedule restore tests and failover drills; measure actual RPO and RTO |

## Hands-on

1. On the Azure SQL lab database, insert rows, note the time, delete them, and restore to the earlier point in time. Compare row counts.
2. Configure long-term retention with weekly backups kept 12 weeks:

```bash
az sql db ltr-policy set -g $RG -s $SRV -n $DB --weekly-retention P12W
```

3. If budget allows, create a failover group to a second region, connect through the listener, perform a planned failover, and confirm your application reconnects.
4. Write a one-page DR runbook for one database: triggers, decision maker, steps, verification, communication, failback.

## Security, cost and reliability notes

- **Security:** backups contain all your data; protect access to restore operations with RBAC and PIM, and use CMK if required.
- **Security:** a compromised admin can delete databases and replicas; use delete locks, separate backup administration and immutable copies.
- **Cost:** geo-redundant backup storage, secondary regions and long retention all add cost. Size them to requirements.
- **Reliability:** untested backups and failovers are assumptions. Test, measure and record.
- **Reliability:** after a forced failover, check for lost transactions and reconcile.

## Common mistakes

- Treating geo-replication as backup.
- No restore tests until an incident.
- DR for the database but not for the application, identity, secrets or DNS.
- Apps with hard-coded server names instead of failover group listeners.
- PITR retention shorter than the time it takes to discover corruption.
- Automatic failover enabled without understanding when it triggers.
- Restoring over production instead of to a new database and validating first.

## Check yourself

1. Define RPO and RTO.
2. What does zone-redundant HA protect against, and what does it not protect against?
3. Compare active geo-replication and failover groups for Azure SQL.
4. What is the DR option for PostgreSQL flexible server with the lowest RPO?
5. What does Cosmos DB continuous backup provide?
6. Why is customer-managed failover usually preferred?
7. Name four things besides the database that must be available in the DR region.

<details>
<summary>Answers</summary>

1. RPO is the maximum acceptable data loss measured in time; RTO is the maximum acceptable time to restore service.
2. It protects against node and zone failures within a region; it does not protect against regional outages or against data deletion and corruption.
3. Active geo-replication: per database, up to four readable secondaries, failover changes the endpoint. Failover groups: groups of databases with listener endpoints that automatically point to the current primary, simplifying failover for applications.
4. A cross-region read replica, promoted on failure.
5. Point-in-time restore to any second within 7 or 30 days, to a new account.
6. It gives you control and speed: you decide based on your own criteria and runbooks rather than waiting for a broad Microsoft decision.
7. Any four of: application compute, identity and secrets (Key Vault), networking and DNS, configuration, caches, search indexes, messaging, monitoring, and the people and runbooks to operate it.

</details>

## References

- [Business continuity in Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/business-continuity-high-availability-disaster-recover-hadr-overview)
- [Automated backups in Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview)
- [Failover groups](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db)
- [PostgreSQL flexible server business continuity](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-business-continuity)
- [Cosmos DB high availability](https://learn.microsoft.com/azure/reliability/reliability-cosmos-db-nosql)
- [Cosmos DB continuous backup](https://learn.microsoft.com/azure/cosmos-db/continuous-backup-restore-introduction)
- [Reliability in Azure Managed Redis](https://learn.microsoft.com/azure/redis/high-availability)
- [Recommendations for disaster recovery (Well-Architected)](https://learn.microsoft.com/azure/well-architected/reliability/disaster-recovery)
