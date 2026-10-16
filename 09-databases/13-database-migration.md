# Database Migration

> Topic: 09 · Databases · Tracks: E – A ● C ● · Status checked: 2026-10

## Why it matters

Database migration is often the critical path of a move to Azure: applications can be redeployed quickly, but data has to be moved consistently, with acceptable downtime and without loss. Consultants scope and estimate these migrations; architects choose targets and cutover strategies; engineers run them. Getting the assessment and cutover right is what separates a quiet weekend from a failed go-live.

## Key concepts

### The migration process

```text
1. Discover     inventory instances, databases, sizes, versions, dependencies, workloads
2. Assess       compatibility with targets, performance baselines, SKU recommendations, cost
3. Plan         target per database, method (offline / online), cutover window, rollback
4. Migrate      schema, data, logins and permissions, jobs, linked objects
5. Validate     row counts, checksums, application tests, performance tests
6. Cut over     stop writes, final sync, switch connection strings, smoke tests
7. Optimise     right-size, tune, enable HA, DR, monitoring, decommission source
```

### Offline vs online migration

| | Offline | Online (minimal downtime) |
|---|---|---|
| Method | Stop the application, copy, switch | Initial load, then continuous replication of changes, then a short cutover |
| Downtime | Duration of the copy | Minutes |
| Complexity | Low | Higher: replication prerequisites, monitoring lag |
| Fit | Small databases, generous windows | Large databases, business-critical systems |

### Tools by source and target

| Source → Target | Assessment | Migration |
|---|---|---|
| SQL Server → Azure SQL (DB, MI, VM) | **Azure Migrate** (discovery and SQL assessment at scale), Azure Arc-enabled SQL Server migration assessment | **Azure Database Migration Service (DMS)** online and offline; **Managed Instance link** (online to MI); backup and restore to MI or VM; log shipping; BACPAC for small DBs |
| PostgreSQL → Azure PostgreSQL flexible | Built-in migration service pre-checks | **Migration service in Azure Database for PostgreSQL** (online and offline, from on-premises, VMs, AWS RDS and Aurora, Google Cloud SQL, Single Server); `pg_dump`/`pg_restore` |
| MySQL → Azure MySQL flexible | – | **DMS** (online and offline); dump and restore plus data-in replication; MySQL import |
| Oracle → PostgreSQL | Assessment tools in the Azure PostgreSQL migration tooling, partner tools | Schema conversion (with AI-assisted conversion tools; check current offerings), Ora2Pg, data migration tools |
| Oracle → Oracle Database@Azure | Oracle tools | Oracle Zero Downtime Migration, Data Guard, RMAN |
| MongoDB → Azure DocumentDB | DocumentDB migration extension for VS Code | Same extension (online and offline); `mongodump`/`mongorestore` |
| MongoDB / Cassandra → Cosmos DB | Compatibility checks | Partner and Spark-based tools, Cosmos DB desktop data migration tool |
| Azure Cache for Redis → Azure Managed Redis | SKU mapping guidance | Rebuild cache, dual-write, or RDB export and import (file 08) |

Older tools have been retired or replaced (for example, Azure Data Studio was retired in 2026 and the Data Migration Assistant is superseded). Use Azure Migrate, the Azure portal DMS experience, Arc-enabled SQL Server assessments and the VS Code extensions. Check current tool status before planning.

### What moves besides data

| Item | Notes |
|---|---|
| Schema, indexes, constraints | Usually migrated first; validate unsupported features |
| Logins, users, permissions | Map to Entra identities where possible; recreate roles |
| SQL Agent jobs, maintenance plans | MI supports SQL Agent; for SQL Database use Elastic Jobs, Functions or Data Factory |
| Linked servers, cross-database queries | Supported on MI; redesign for SQL Database |
| Encryption keys, certificates | TDE certificates must be migrated for encrypted backups |
| Application connection strings | Centralise in configuration (App Configuration, Key Vault) to cut over cleanly |
| Monitoring and alerts | Rebuild in Azure Monitor |
| ETL and reporting | Repoint pipelines and reports |

### Cutover strategy

```text
T-2 weeks   initial load complete; continuous replication running; validation passing
T-1 week    performance test against target; rehearsal of cutover; rollback plan agreed
T-0         announce freeze → stop application writes → wait for replication lag = 0
            → final validation (row counts, checksums) → switch connection strings
            → smoke tests → reopen application
T+1 day     monitor performance and errors; keep source read-only for rollback window
T+2 weeks   decommission source
```

Rollback options: keep the source intact and read-only; for longer rollback windows, set up reverse replication (for example the MI link supports failback to SQL Server).

## How it works on Azure

### Sizing from performance data

Assessment tools collect CPU, memory, IOPS, throughput and storage over a representative period (at least a week, including month-end). The recommendation should be based on percentiles (for example P95), not peaks or averages alone. Validate with a load test on the target.

### Validation queries

```sql
-- Row counts per table (run on source and target, compare)
SELECT s.name AS schema_name, t.name AS table_name, SUM(p.rows) AS row_count
FROM sys.tables t
JOIN sys.schemas s ON t.schema_id = s.schema_id
JOIN sys.partitions p ON t.object_id = p.object_id AND p.index_id IN (0,1)
GROUP BY s.name, t.name
ORDER BY s.name, t.name;

-- Aggregate checksum for a critical table
SELECT CHECKSUM_AGG(BINARY_CHECKSUM(*)) FROM dbo.Orders;
```

### Estimating effort (consultant's view)

| Factor | Raises effort |
|---|---|
| Number and size of databases | Many small databases need automation; very large ones need online methods |
| Compatibility issues | Each blocker needs remediation or a different target |
| Heterogeneous migration (Oracle → PostgreSQL) | Code conversion (stored procedures, PL/SQL), testing, retraining; often the largest effort |
| Downtime constraints | Online methods, rehearsals |
| Application coupling | Many apps sharing a database require coordinated cutover |
| Compliance | Data handling approvals, encryption key management, audit evidence |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Rehost vs replatform vs refactor | Rehost (SQL on VM) for speed; replatform (MI, flexible servers) for managed benefits with few changes; refactor (SQL Database, Cosmos DB) when modernising the app |
| Target choice | Follow file 01 and the assessment results; MI is the usual landing zone for complex SQL Server estates |
| Online or offline | Offline if the window allows; online for large or critical databases |
| Wave planning | Group databases by application and dependency; start with low-risk waves to build confidence |
| Heterogeneous migrations | Only with clear business value (licence cost, strategy); budget for code conversion and testing |
| Licensing | Apply Azure Hybrid Benefit; consider free ESUs on Azure VMs to buy time |

## Hands-on

1. **Assessment exercise:** for a sample SQL Server estate (or the AdventureWorks sample on a local SQL Server), run an Azure Migrate or Arc-based SQL assessment, and record the recommended target and SKU for each database with blockers.
2. **PostgreSQL migration:** create a source PostgreSQL (a container on your laptop or a VM), load sample data, and migrate it to a flexible server with `pg_dump`/`pg_restore`. Then repeat using the built-in migration service in offline mode.

```bash
pg_dump -h localhost -U postgres -Fc -d sampledb -f sampledb.dump
export PGPASSWORD=$(az account get-access-token --resource-type oss-rdbms --query accessToken -o tsv)
pg_restore -h $PG.postgres.database.azure.com -U $ME_UPN -d sampledb --no-owner --no-acl sampledb.dump
```

3. Write a cutover runbook for the migration including validation queries and a rollback step.

## Security, cost and reliability notes

- **Security:** migration paths carry full copies of data; use private connectivity (VPN or ExpressRoute with private endpoints), encryption in transit, and least-privilege migration identities. Remove temporary access afterwards.
- **Security:** take the opportunity to move from SQL logins to Entra identities.
- **Cost:** run source and target in parallel only as long as needed; DMS premium SKUs for online migrations have costs.
- **Cost:** right-size after migration from real usage; assessment recommendations are a starting point.
- **Reliability:** rehearse cutovers; have tested rollback; keep the source available until the rollback window closes.

## Common mistakes

- Skipping assessment and discovering blockers on cutover night.
- Sizing from average utilisation, then poor performance at month-end.
- Forgetting logins, jobs, linked servers or encryption certificates.
- No validation beyond "the application starts".
- Connection strings scattered across many configuration files, making cutover slow and error-prone.
- Heterogeneous migrations estimated as if they were like-for-like.
- Using retired tools from old tutorials.

## Check yourself

1. List the seven stages of a database migration.
2. When would you choose online over offline migration?
3. Which tools would you use for SQL Server → Managed Instance with minimal downtime?
4. What tool migrates from AWS RDS for PostgreSQL to Azure PostgreSQL flexible server?
5. What objects besides data must be migrated for SQL Server?
6. How do you validate that data migrated correctly?
7. What makes Oracle → PostgreSQL migrations expensive?

<details>
<summary>Answers</summary>

1. Discover, assess, plan, migrate, validate, cut over, optimise.
2. When the database is too large for the downtime window, or the system is business-critical and can tolerate only minutes of downtime.
3. Azure Migrate or Arc-based assessment, then the Managed Instance link or DMS in online mode.
4. The migration service in Azure Database for PostgreSQL (online or offline), or `pg_dump`/`pg_restore` for offline.
5. Logins and users, permissions, SQL Agent jobs, linked servers, certificates and keys, maintenance plans, plus ETL and reports that depend on the database.
6. Compare row counts and checksums between source and target, run application tests and performance tests before cutover.
7. Converting schema and procedural code (PL/SQL), differences in data types and behaviour, extensive testing, and retraining teams.

</details>

## References

- [Database migration guides](https://learn.microsoft.com/data-migration/)
- [Azure Database Migration Service](https://learn.microsoft.com/azure/dms/dms-overview)
- [Azure Migrate: SQL Server assessment](https://learn.microsoft.com/azure/migrate/concepts-azure-sql-assessment-calculation)
- [Migration assessment for Arc-enabled SQL Server](https://learn.microsoft.com/sql/sql-server/azure-arc/migration-overview)
- [Migration service in Azure Database for PostgreSQL](https://learn.microsoft.com/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)
- [Migrate MongoDB to Azure DocumentDB](https://learn.microsoft.com/azure/documentdb/migration-options)
- [Oracle to Azure PostgreSQL migration guide](https://learn.microsoft.com/azure/postgresql/migrate/oracle-to-postgres/oracle-to-azure-postgresql-migration-overview)
- [Cloud Adoption Framework: migrate](https://learn.microsoft.com/azure/cloud-adoption-framework/migrate/)
