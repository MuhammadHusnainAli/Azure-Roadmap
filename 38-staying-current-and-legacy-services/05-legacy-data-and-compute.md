# Legacy Data and Compute

## Why it matters

AI initiatives depend on data platforms, and many enterprises still run data and compute services from earlier Azure generations. Common examples are Azure Synapse Analytics, HDInsight, Azure Analysis Services, and even retired services such as Data Lake Analytics and classic (ASM) resources. When you plan RAG, analytics or agent solutions, you'll find these underneath. You need to know their status, how they connect to modern AI services, and how to migrate without disrupting the business. Microsoft's strategic analytics platform is now **Microsoft Fabric**, and that shapes most modernisation recommendations.

## Key concepts

### Status and direction

| Service | Status (at time of writing) | Direction for new work |
|---|---|---|
| **Azure Synapse Analytics** (dedicated SQL pools, serverless SQL, Spark pools, pipelines, Data Explorer pools) | Supported; Microsoft positions **Fabric** as the platform for new analytics and provides Synapse-to-Fabric migration guidance | Fabric Warehouse, Lakehouse, Data Factory in Fabric, Real-Time Intelligence |
| **Azure SQL Data Warehouse** (Gen1, Gen2 naming) | Became Synapse dedicated SQL pools | Fabric Warehouse for new work |
| **HDInsight** | HDInsight 5.1 supported; 5.0 and earlier retired (5.0 on 31 March 2025) | Fabric Spark, Azure Databricks, or HDInsight 5.1 for lift-and-shift |
| **Azure Data Lake Analytics (U-SQL)** | **Retired 29 February 2024** | Synapse or Fabric Spark, Databricks |
| **Azure Data Lake Storage Gen1** | **Retired 29 February 2024** | Data Lake Storage (Gen2, hierarchical namespace) |
| **Azure Analysis Services** | Supported; some older SKUs deprecated | Power BI or Fabric semantic models (Direct Lake) for new work |
| **Azure Time Series Insights** | **Retired 7 July 2024** | Azure Data Explorer, Fabric Real-Time Intelligence |
| **Azure Database for MariaDB** | Retired (2025) | Azure Database for MySQL flexible server |
| **Single Server** (MySQL and PostgreSQL) | Retired | Flexible server |
| **Classic (ASM) VMs, storage, networking** | Retired | Azure Resource Manager equivalents |
| **Cloud Services (extended support)** | Retires 31 March 2027 | VM scale sets, App Service, Container Apps, AKS |
| **Azure Batch** | Supported | Still valid for HPC and batch; consider Container Apps jobs for container workloads |

See [Topic 12](../12-data-and-analytics/README.md) for the modern data platform and [Topic 30](../30-migration-and-modernisation/README.md) for migration tooling.

### Connecting legacy data to AI

| Legacy source | AI integration options before migration |
|---|---|
| Synapse dedicated SQL pool | Fabric mirroring or shortcuts; export to Data Lake Storage for indexing; Foundry agents via Fabric data agents once data is in Fabric |
| HDInsight output in Data Lake Storage | OneLake shortcuts; AI Search indexers on Blob/Data Lake Storage |
| Analysis Services models | Migrate to Power BI or Fabric semantic models to use Copilot and Fabric data agents |
| SQL Server on VMs | Azure SQL vector support, Fabric mirroring, AI Search indexers |

Often the quickest AI win is to **expose legacy data through OneLake shortcuts or mirroring** and leave the full platform migration for later.

## How it works on Azure

### Modernisation approach

1. **Inventory** data platforms, pipelines, consumers and SLAs.
2. **Classify** each item by status (retired, retiring, legacy-supported, current) and by business value.
3. **Choose a pattern** per workload:
   - **Bridge**: shortcuts or mirroring into Fabric for new analytics and AI; the legacy platform stays.
   - **Lift and shift**: HDInsight 5.0 to 5.1, Single Server to flexible server.
   - **Re-platform**: Synapse dedicated SQL pool to Fabric Warehouse; Spark jobs to Fabric or Databricks.
   - **Retire**: unused pipelines and reports.
4. **Migrate in waves** with parallel runs and data reconciliation.
5. **Decommission** with access reviews and cost validation.

### Example: Synapse to Fabric (outline)

```text
Assess: dedicated pools, serverless SQL, Spark, pipelines, consumers
Bridge: create Fabric workspace; OneLake shortcuts to existing ADLS; mirror where supported
Migrate: schema + code (T-SQL differences) → Fabric Warehouse; pipelines → Data Factory in Fabric
Validate: row counts, checksums, report parity, performance
Cut over consumers; pause and then delete Synapse pools
```

Follow Microsoft's Synapse-to-Fabric migration guidance for supported tooling and T-SQL differences.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Fabric vs Databricks** | Fabric for Microsoft-centric analytics and Power BI integration; Databricks for advanced data engineering, ML and multicloud lakehouse teams; many enterprises use both |
| **Bridge vs migrate** | Bridge for speed to AI value; migrate for cost, simplicity and retirement risk |
| **Capacity model** | Fabric capacity (F SKUs) differs from per-service billing; model costs carefully |
| **Skills** | T-SQL teams adapt quickly to Fabric Warehouse; Hadoop-era skills need retraining |

## Hands-on

1. Inventory data services in a subscription with Resource Graph and classify them by status.
2. Create a OneLake shortcut to an existing Data Lake Storage container and query it.
3. Draft a Synapse-to-Fabric migration outline for one workload.
4. Identify any HDInsight clusters on retired versions and plan the upgrade.
5. Estimate cost before and after for one modernisation.

## Security, cost and reliability notes

- Retired platforms no longer receive security updates. Prioritise them.
- Paused Synapse dedicated pools still incur storage costs; delete them after migration.
- Keep data reconciliation evidence for auditors during migration.
- Apply Purview classification and access policies before exposing data to AI.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new AI on retired or legacy platforms | Bridge to Fabric or modern stores |
| Big-bang data migration | Waves with parallel runs |
| Ignoring downstream consumers | Map reports, APIs and extracts |
| Exposing data to AI without governance | Purview classification and access control first |

## Check yourself

1. What is Microsoft's strategic platform for new analytics workloads?
2. When did Data Lake Analytics and Data Lake Storage Gen1 retire?
3. What's the fastest way to make legacy lake data available to Fabric?
4. What's the direction for Analysis Services models?
5. Name four modernisation patterns.

<details>
<summary>Answers</summary>

1. Microsoft Fabric.
2. 29 February 2024.
3. OneLake shortcuts (or mirroring where supported).
4. Power BI or Fabric semantic models.
5. Bridge, lift and shift, re-platform, retire.

</details>

## References

- [Migrate Synapse dedicated SQL pools to Fabric Warehouse](https://learn.microsoft.com/fabric/data-warehouse/migration-synapse-dedicated-sql-pool-warehouse)
- [OneLake shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-shortcuts)
- [HDInsight component versioning](https://learn.microsoft.com/azure/hdinsight/hdinsight-component-versioning)
- [Migrate Data Lake Analytics to Synapse](https://learn.microsoft.com/previous-versions/azure/data-lake-analytics/migrate-azure-data-lake-analytics-to-synapse)
- [What is Azure Synapse Analytics?](https://learn.microsoft.com/azure/synapse-analytics/overview-what-is)
