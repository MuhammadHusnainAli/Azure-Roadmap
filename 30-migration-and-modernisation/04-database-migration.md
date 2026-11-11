# Database Migration

## Why it matters

Databases are where migrations get hard: they hold the data the business can't lose, they're shared by many applications, they need consistent cutovers with minimal downtime, and their target choice (IaaS SQL Server, Azure SQL Managed Instance, Azure SQL Database, PostgreSQL, Oracle Database@Azure) sets licensing costs and operational effort for years. This file focuses on the **programme view**: choosing target paths by source engine, sequencing database moves within migration waves, and coordinating application cutover. Tool-level detail and validation queries are in [Topic 09 file 13](../09-databases/13-database-migration.md).

## Key concepts

### Target paths by source

| Source | Rehost | Replatform (PaaS) | Modernise |
|---|---|---|---|
| **SQL Server** | SQL Server on Azure VMs (full compatibility, AHB, ESU) | **Azure SQL Managed Instance** (near-full compatibility: SQL Agent, cross-database queries, CLR) | **Azure SQL Database** (Hyperscale/serverless, best for new or refactored apps) |
| **Oracle** | Oracle on Azure VMs | **Oracle Database@Azure** (Oracle Exadata/Autonomous services in Azure datacentres) | Convert to **PostgreSQL** (schema conversion, code rewrite of PL/SQL) |
| **PostgreSQL** | PostgreSQL on VMs | **Azure Database for PostgreSQL flexible server** | Same, plus vector/AI extensions |
| **MySQL** | MySQL on VMs | **Azure Database for MySQL flexible server** | — |
| **MongoDB** | — | **Azure DocumentDB** (MongoDB compatible) or Cosmos DB for MongoDB | Cosmos DB NoSQL redesign |
| **DB2, Sybase, others** | VMs | Partner tools to Azure SQL/PostgreSQL | Rearchitect |

### Tools summary

| Need | Tool |
|---|---|
| Discovery and SQL assessment at scale | **Azure Migrate** (SQL instances discovered by the appliance; Azure SQL assessments), Arc-enabled SQL Server assessments |
| SQL Server online/offline migration | **Azure Database Migration Service (DMS)**; **Managed Instance link** (near-real-time replication to SQL MI, also for DR/hybrid); backup/restore; log shipping |
| PostgreSQL | **Migration service in Azure Database for PostgreSQL** (online/offline from on-premises, VMs, AWS RDS/Aurora, Google Cloud SQL) |
| MySQL | DMS, MySQL import, dump/restore with data-in replication |
| Oracle | Oracle Zero Downtime Migration/Data Guard to Oracle Database@Azure; conversion tooling and Ora2Pg for PostgreSQL |
| MongoDB | DocumentDB migration extension for VS Code |

Azure Data Studio was retired in 2026; use Azure Migrate, the portal DMS experience and VS Code extensions.

### Online vs offline

| Mode | Downtime | Complexity |
|---|---|---|
| **Offline** | Duration of final copy and restore | Simpler; fine for small DBs or generous windows |
| **Online** | Minutes (cutover after continuous sync) | Requires replication/CDC support, monitoring lag |

### Programme considerations

| Area | Guidance |
|---|---|
| **Shared databases** | Migrate with all dependent apps in the same wave, or provide temporary hybrid connectivity |
| **Server objects** | Logins, jobs, linked servers, SSIS/SSRS, CLR, certificates, collation, time zones |
| **Performance baseline** | Capture before migration; compare after (Query Store) |
| **Compatibility level** | Keep source level initially; upgrade after stabilisation |
| **Licensing** | AHB for SQL Server; Oracle licensing terms for Database@Azure vs VMs |
| **Security** | Entra authentication, private endpoints, TDE/CMK, auditing on arrival |

## How it works on Azure

Wave-level database cutover runbook (online SQL Server → SQL MI):

```text
T-14d  Assessment issues resolved; target MI provisioned (zone-redundant, private endpoint, Entra admin)
T-7d   Start online migration (DMS or MI link); monitor replication lag
T-2d   Application tested against a read-only replica/snapshot; performance compared with baseline
T-0    Change freeze → stop app writes → wait for lag = 0 → cutover → repoint connection strings (or DNS alias)
       → smoke tests → reopen traffic
T+1d   Monitor Query Store and errors; keep source read-only for rollback window
T+14d  Decommission source after sign-off
```

Rollback: if validation fails before reopening writes, repoint to the (still intact) source; after writes resume on the target, rollback requires reverse replication (MI link supports failback scenarios) or data reconciliation, so decide the point of no return explicitly.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **MI vs SQL DB vs VM** | MI for lift-and-shift of instance-level features; SQL DB for modern apps; VMs only when OS access or unsupported features are required. |
| **Oracle** | Oracle Database@Azure for minimal change and Oracle-specific features; PostgreSQL conversion for long-term licence savings with significant effort. |
| **Downtime tolerance** | Drives online vs offline; online increases complexity. |
| **Sequence** | Databases with many consumers early or with hybrid connectivity; isolated databases with their apps. |
| **Consolidation** | Elastic pools or MI instance pools for many small databases. |

## Hands-on

1. Run an Azure SQL assessment for discovered SQL Server instances; review MI vs SQL DB readiness.
2. Perform an online migration of a sample database to SQL MI with DMS or MI link; measure cutover downtime.
3. Migrate a PostgreSQL database with the PostgreSQL migration service.
4. Write a cutover and rollback runbook with a point of no return.
5. Compare Query Store performance before and after migration.

## Security, cost and reliability notes

- **Credentials and network** for migration tools: least privilege, private connectivity, remove after migration.
- **Data in transit**: encrypt; avoid public endpoints.
- **Cost**: target databases run in parallel with sources during migration; plan the overlap.
- **Reliability**: configure zone redundancy, backups and failover groups as part of the target (Topic 28 file 05).

## Common mistakes

| Mistake | Fix |
|---|---|
| Migrating a shared database without its consumers | Wave by dependency |
| Forgetting jobs, logins, linked servers | Object inventory |
| No performance baseline | Capture before migration |
| Unclear rollback point | Define point of no return |
| Using retired tools | Current tools per docs |

## Check yourself

1. Compare SQL Server on VMs, SQL MI and SQL Database as targets.
2. What are the options for Oracle?
3. Which tools migrate SQL Server, PostgreSQL and MongoDB?
4. When is online migration worth the complexity?
5. What non-data objects must move?
6. What is the point of no return in a database cutover?

<details>
<summary>Answers</summary>

1. VMs: full compatibility and OS access; MI: near-full instance compatibility as PaaS; SQL DB: modern, most managed, for new or refactored apps.
2. Oracle on VMs, Oracle Database@Azure, or conversion to PostgreSQL.
3. SQL Server: Azure Migrate assessments, DMS, MI link, backup/restore. PostgreSQL: Azure PostgreSQL migration service. MongoDB: DocumentDB migration extension for VS Code.
4. When downtime must be minutes and the source supports continuous replication.
5. Logins, jobs, linked servers, SSIS/SSRS, CLR, certificates, collation and time zone settings.
6. The moment writes resume on the target, after which rollback needs reverse replication or reconciliation.

</details>

## References

- [Azure SQL migration guides](https://learn.microsoft.com/azure/azure-sql/migration-guides/)
- [Azure Database Migration Service](https://learn.microsoft.com/azure/dms/dms-overview)
- [Managed Instance link](https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [PostgreSQL migration service](https://learn.microsoft.com/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)
- [Oracle Database@Azure](https://learn.microsoft.com/azure/oracle/oracle-db/database-overview)
