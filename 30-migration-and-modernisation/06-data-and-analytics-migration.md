# Data and Analytics Migration

## Why it matters

Data platforms are central to most cloud and AI programmes: you can't build reliable RAG, agents or analytics on top of fragmented on-premises warehouses, nightly SSIS jobs and spreadsheets. Moving warehouses, lakes and pipelines to **Microsoft Fabric** or **Azure Databricks** consolidates data, reduces infrastructure management and opens it to AI. Data migrations are also among the riskiest: reports that finance depends on must produce identical numbers, pipelines have hidden dependencies, and data volumes make cutovers slow. This file covers target choices, migration paths and tooling, especially Fabric's migration assistants and upgrade paths from Azure Synapse, Azure Data Factory and Analysis Services.

## Key concepts

### Target platforms

| Platform | Strengths | Fit |
|---|---|---|
| **Microsoft Fabric** | SaaS, unified OneLake, Data Factory, Data Engineering (Spark), Data Warehouse (T-SQL), Real-Time Intelligence, Power BI, Copilot; capacity-based pricing | Microsoft-centric organisations, Power BI estates, SQL-skilled teams, consolidation ([Topic 12](../12-data-and-analytics/README.md)) |
| **Azure Databricks** | Lakehouse with Delta Lake, Unity Catalog, strong Spark/ML/data engineering, multi-cloud | Data engineering-heavy teams, ML platforms, existing Databricks/Spark skills |
| **Both together** | OneLake shortcuts and mirroring of Databricks Unity Catalog; Databricks for engineering, Fabric/Power BI for consumption | Common enterprise pattern |
| **Azure SQL/PostgreSQL** | Operational databases | Smaller analytical needs, application databases (file 04) |

### Migration sources and paths

| Source | Path |
|---|---|
| **Azure Synapse dedicated SQL pools** | **Migration Assistant for Fabric Data Warehouse** (moves objects and data; check supported scenarios); planning and methods guidance |
| **Azure Synapse Spark** | Synapse Spark → Fabric migration guidance (notebooks, Spark job definitions, pools, metadata) |
| **Azure Synapse Data Explorer** | → Fabric Eventhouse (Preview) |
| **Azure Data Factory** | Upgrade paths to Fabric Data Factory; **mount ADF in Fabric** (Azure Data Factory item) for immediate visibility; rebuild pipelines over time |
| **Azure Analysis Services** | → Power BI semantic models (Power BI Premium/Fabric migration feature) |
| **On-premises SQL Server warehouses / SSIS** | Data Factory pipelines or SSIS in Fabric/ADF (Azure-SSIS IR), warehouse rebuild in Fabric or Databricks |
| **Teradata, Netezza, Oracle Exadata DW** | Partner tools and schema conversion; data via ADF/Fabric pipelines |
| **Hadoop/HDInsight** | Databricks or Fabric Spark; HDFS data to ADLS/OneLake |
| **Third-party BI** | Migrate to Power BI with planning guidance |

### Approaches

| Approach | Description |
|---|---|
| **Lift and shift** | Same schema and logic on the new platform; fastest; carries technical debt |
| **Modernise during migration** | Medallion architecture (bronze/silver/gold), Delta/Parquet, semantic models; more value, more effort |
| **Hybrid coexistence** | Mirroring/shortcuts to make existing data available in Fabric immediately, then migrate workloads gradually |

### Key workstreams

Schema and code conversion (T-SQL dialect differences, stored procedures), data movement (initial load + incremental), pipeline migration, semantic layer and reports, security model (row-level security, workspace roles, Purview), validation (row counts, checksums, report parity), cutover and decommission.

## How it works on Azure

Synapse dedicated SQL pool → Fabric Data Warehouse (outline):

```text
1. Inventory: tables, views, stored procedures, workloads, data volumes, dependent pipelines and reports.
2. Plan: choose methods (Migration Assistant for objects/data, pipelines/COPY for large tables, CETAS/Parquet export).
3. Migration Assistant: connect source, convert schema (review conversion issues), migrate objects, copy data.
4. Repoint pipelines (ADF → Fabric Data Factory or mounted ADF) and Power BI semantic models.
5. Validate: row counts, aggregates, report parity for key finance reports.
6. Run in parallel for a reporting cycle; cut over consumers; pause and decommission Synapse.
```

Validation query pattern (run on both source and target):

```sql
SELECT 'FactSales' AS tbl, COUNT_BIG(*) AS rows, SUM(CAST(SalesAmount AS DECIMAL(38,2))) AS total_sales,
       MIN(OrderDate) AS min_date, MAX(OrderDate) AS max_date
FROM dbo.FactSales;
```

Coexistence with mirroring: mirror Azure SQL, Cosmos DB, Snowflake or other supported sources into OneLake for near-real-time analytics without building pipelines, then migrate transformations at your own pace ([Topic 12](../12-data-and-analytics/README.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Fabric vs Databricks** | Fabric for integrated SaaS and Power BI-centric organisations; Databricks for engineering/ML depth and multi-cloud; both via OneLake integration is common. |
| **Big bang vs incremental** | Incremental by subject area/data product; keep finance-critical reports in parallel for at least one close cycle. |
| **Modernise or lift** | Lift where deadlines force it; modernise high-value domains into medallion and semantic models. |
| **Capacity sizing** | Fabric capacity (F SKUs) sized from workload tests; reservations once steady (Topic 29). |
| **Governance** | Purview, Unity Catalog or Fabric domains defined before migrating sensitive data. |
| **AI readiness** | Curated, governed gold layers become grounding sources for Fabric data agents and Foundry IQ. |

## Hands-on

1. Inventory a Synapse dedicated SQL pool and plan migration methods per table size.
2. Use the Migration Assistant for Fabric Data Warehouse on a sample database; review conversion issues.
3. Mount an existing Azure Data Factory in Fabric and plan pipeline upgrades.
4. Migrate an Azure Analysis Services model to Power BI.
5. Build a validation workbook comparing row counts and totals between source and target.

## Security, cost and reliability notes

- **Security model parity**: row-level security, object permissions and sensitivity labels must be recreated and tested.
- **Data residency**: Fabric capacity and OneLake region; Databricks workspace region.
- **Cost**: parallel running of old and new platforms; Fabric capacity smoothing and pause/resume for non-production.
- **Reliability**: backup/retention and DR for the new platform (OneLake, Delta time travel, workspace recovery).

## Common mistakes

| Mistake | Fix |
|---|---|
| Migrating without report parity checks | Validation and parallel run |
| Ignoring stored procedure dialect differences | Conversion review |
| Moving everything at once | Incremental by domain |
| No governance model in the target | Purview/Unity Catalog/domains first |
| Underestimating pipeline dependencies | Inventory and lineage |

## Check yourself

1. When do you choose Fabric, Databricks or both?
2. What tooling helps migrate Synapse dedicated SQL pools?
3. What options exist for Azure Data Factory pipelines?
4. Describe three migration approaches.
5. How do you validate a warehouse migration?
6. How can mirroring support coexistence?

<details>
<summary>Answers</summary>

1. Fabric for integrated SaaS analytics and Power BI-centric teams; Databricks for engineering/ML depth and multi-cloud; both with OneLake integration for engineering plus consumption.
2. The Migration Assistant for Fabric Data Warehouse plus planning and methods guidance.
3. Upgrade paths to Fabric Data Factory, mounting ADF in Fabric, or keeping ADF and integrating.
4. Lift and shift, modernise during migration, hybrid coexistence.
5. Row counts, aggregates/checksums, date ranges and report parity on both systems, plus a parallel run.
6. It replicates supported sources into OneLake for immediate analytics while transformations migrate gradually.

</details>

## References

- [Microsoft Fabric migration overview](https://learn.microsoft.com/fabric/fundamentals/migration)
- [Migration Assistant for Fabric Data Warehouse](https://learn.microsoft.com/fabric/data-warehouse/migration-assistant)
- [Azure Data Factory to Fabric upgrade paths](https://learn.microsoft.com/fabric/data-factory/upgrade-paths)
- [Migrate Synapse Spark to Fabric](https://learn.microsoft.com/fabric/data-engineering/migrate-synapse-overview)
- [Azure Databricks migration](https://learn.microsoft.com/azure/databricks/migration/)
