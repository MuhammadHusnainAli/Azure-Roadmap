# Azure Data Factory and Azure Synapse Analytics

## Why it matters

Many organisations already run large estates on **Azure Data Factory (ADF)** and **Azure Synapse Analytics**. Microsoft's strategic analytics platform is now **Microsoft Fabric**, and most new investment lands there. That makes the most common consulting question here "should we migrate, when, and how?". Answering well means understanding what these services do, what maps cleanly to Fabric, what doesn't, and which migration tools exist, without overselling a "lift and shift" that isn't one.

## Key concepts

### Azure Data Factory recap

ADF is Azure's PaaS data integration service: pipelines, activities, datasets, linked services, triggers and integration runtimes (Azure, self-hosted, Azure-SSIS). Topic 11 file 09 covers it in detail. ADF remains fully supported and is still the right tool in some cases, for example SSIS lift-and-shift and integration not centred on Fabric.

### Azure Synapse Analytics recap

Synapse is an integrated analytics workspace that combined several engines:

| Component | What it is | Fabric counterpart |
|---|---|---|
| **Dedicated SQL pool** (formerly SQL DW) | Provisioned MPP data warehouse (DWUs), with distributions, indexes and PolyBase | **Fabric Data Warehouse** |
| **Serverless SQL pool** | Pay-per-query T-SQL over files in the lake | **Lakehouse SQL analytics endpoint**, Warehouse, shortcuts |
| **Apache Spark pools** | Managed Spark | **Fabric Data Engineering** (lakehouse, notebooks, environments) |
| **Synapse pipelines** | ADF engine inside Synapse | **Fabric Data Factory** pipelines |
| **Data Explorer pools** (Preview) | Kusto engine inside Synapse | **Eventhouse** (Real-Time Intelligence) or Azure Data Explorer |
| **Synapse Link** (Cosmos DB, SQL, Dataverse) | Near real-time replication to analytics | **Fabric Mirroring**, Dataverse **Link to Fabric** |
| Synapse Studio | Unified authoring | Fabric portal |

Synapse Analytics is still supported, but Microsoft guides new analytics projects to Fabric. Check the Synapse documentation and Azure Updates for any retirement notices that apply to specific components (some preview features, such as Data Explorer pools, have had separate notices).

## How it works on Azure

### Mapping concepts from ADF/Synapse pipelines to Fabric Data Factory

| ADF / Synapse | Fabric Data Factory |
|---|---|
| Linked service + dataset | **Connection** (datasets are gone; settings live in the activity) |
| Pipeline, activities | Pipeline, activities (most are equivalent; some new ones such as Office 365 Outlook, Teams, Semantic model refresh) |
| Mapping data flows | **Dataflows Gen2** (Power Query), or notebooks; a preview upgrade experience exists for mapping data flows |
| Self-hosted IR | **On-premises data gateway**; **virtual network data gateway** for VNet sources |
| Azure IR managed VNet | Managed private endpoints / workspace networking features |
| Triggers (schedule, tumbling, storage, custom event) | Schedules and **event triggers** via Real-Time hub (check tumbling window equivalents) |
| Global parameters | **Variable libraries** |
| ARM-based CI/CD | **Git integration** and **deployment pipelines** |
| Azure-SSIS IR | No direct equivalent; keep in ADF, or invoke SSIS packages from Fabric pipelines where supported (check status) |

### Migration tooling for Data Factory (checked 2026-11)

| Path | What it does | Status |
|---|---|---|
| **Mount ADF in Fabric** (Azure Data Factory item) | Brings an existing factory into a Fabric workspace for side-by-side review and incremental migration of supported pipelines | Check current status |
| **Built-in assessment** in ADF Studio | "Start assessment" analyses pipelines for Fabric compatibility | Preview |
| **PowerShell upgrade module** (`Microsoft.FabricPipelineUpgrade`) | Exports ADF pipelines, maps linked services to Fabric connections with a resolution file, and creates Fabric pipelines. Suited to standard patterns (Copy, Lookup, Stored procedure) at scale | Available; see tutorial |
| **Mapping data flow upgrade** | Converts mapping data flows to Dataflow Gen2 transforms | Preview |
| Manual rebuild | For complex or unsupported patterns | — |

### Migration tooling for Synapse

| Source | Tooling |
|---|---|
| Dedicated SQL pool → Fabric Warehouse | **Migration Assistant for Fabric Data Warehouse**: import a DACPAC, convert the schema, fix failed scripts (with Copilot assistance), copy data, then reroute connections. Plus the planning guide on data type and T-SQL differences |
| Spark pools → Fabric Spark | Migrate notebooks, Spark job definitions, pools (to environments) and Hive metastore metadata; Microsoft provides guidance and scripts. Data often stays in ADLS and is reached through **shortcuts** |
| Serverless SQL → Fabric | Recreate views over shortcuts or lakehouse tables; review OPENROWSET usage |
| Synapse pipelines → Fabric | Same as ADF paths above |
| Synapse Link → Fabric | Replace with Mirroring (Cosmos DB, Azure SQL, SQL Server) or Link to Fabric (Dataverse) |

### Typical differences to plan for (dedicated pool → Fabric Warehouse)

| Area | Change |
|---|---|
| Distribution and indexes | `DISTRIBUTION = HASH(...)`, clustered columnstore and index hints aren't used. Fabric manages layout; remove them |
| Data types | Some types differ or aren't supported (for example `money`, `nvarchar` and some date/time precisions map to alternatives). Review the type mapping table |
| T-SQL surface | Check the T-SQL surface area for unsupported features (for example some DDL, identity behaviour, certain hints) |
| Workload management | Replace resource classes and workload groups with capacity planning, workspaces and separate capacities |
| Security | Map SQL logins and roles to Entra ID, workspace roles, item permissions and T-SQL GRANT |
| Loading | PolyBase external tables → `COPY INTO`, pipelines, shortcuts |
| Cost model | DWU hours → Fabric capacity CUs (shared with other workloads) |

## Design decisions and trade-offs

### Should we migrate?

| Signal | Lean towards |
|---|---|
| Power BI is the main consumer; want Direct Lake, Copilot, data agents | Migrate to Fabric |
| Mostly SSIS packages, or ADF orchestrating non-analytics integration | Keep ADF |
| Heavy dedicated pool with stable workloads and no new requirements | Migrate opportunistically (when contracts or hardware refresh force decisions); no urgency unless a retirement notice applies |
| Need for features only in Fabric (OneLake shortcuts and mirroring, Real-Time Intelligence, Fabric IQ) | Migrate the relevant parts |
| Strict network isolation requirements | Validate Fabric private link and outbound protection features against requirements first |

### Migration approaches

| Approach | Description | When |
|---|---|---|
| **Coexist, then move** | Point Fabric at existing ADLS data with shortcuts; build new workloads in Fabric; retire Synapse components gradually | Most organisations; lowest risk |
| **Lift and adapt** | Migrate warehouse schema and data with the Migration Assistant; upgrade pipelines with the PowerShell module | Well-understood estates with standard patterns |
| **Re-architect** | Redesign into a medallion lakehouse, adopt Direct Lake and data products | When the existing design is the problem |

### Planning steps

1. **Inventory** pipelines, data flows, SQL objects, Spark code, consumers and SLAs (use the ADF assessment and DMV queries on the pool).
2. **Classify** each asset: move as-is, adapt, rebuild, retire.
3. **Size capacity** using a proof of concept and the Capacity Metrics app. Don't map DWUs to CUs mechanically.
4. **Pilot** one domain end to end, including security and CI/CD.
5. **Run in parallel** and reconcile results (row counts, aggregates) before cutting over.
6. **Cut over** consumers (Power BI datasets and apps) and decommission.

## Hands-on

1. Take an ADF factory with two pipelines (one Copy, one with a Lookup and ForEach). Run the built-in **assessment** and review the findings.
2. Use the **PowerShell upgrade module** tutorial to migrate the Copy pipeline to a Fabric workspace, mapping its linked service to a Fabric connection.
3. Export a small Synapse dedicated pool (or SQL Server) schema as a **DACPAC** and import it with the **Migration Assistant**; fix any failed scripts.
4. Create a lakehouse **shortcut** to the ADLS container used by Synapse Spark and query the same Delta tables from a Fabric notebook.
5. Write a one-page migration recommendation for a fictional client using the decision table above.

## Security, cost and reliability notes

- **Security:** re-implement security, don't assume it carries over. SQL logins, managed VNets and self-hosted IRs have different counterparts in Fabric.
- **Cost:** running Synapse and Fabric in parallel doubles some costs. Time-box the parallel run. Pause dedicated SQL pools and Fabric capacities when not needed.
- **Cost:** compare total cost, not unit prices: Fabric capacity is shared by engineering, warehousing, BI and AI.
- **Reliability:** validate data parity with automated reconciliation; keep rollback (the old pipelines) until consumers sign off.

## Common mistakes

- Promising a fully automated migration; complex pipelines, mapping data flows and SSIS need manual work.
- Keeping distribution and index hints in migrated T-SQL.
- Copying all data from ADLS into OneLake instead of using shortcuts.
- Sizing Fabric capacity by translating DWUs directly.
- Migrating everything at once instead of domain by domain.
- Ignoring Power BI dataset re-pointing and consumer communication.

## Check yourself

1. Which Fabric item replaces a Synapse dedicated SQL pool?
2. What replaces ADF linked services and datasets in Fabric Data Factory?
3. What does the Migration Assistant for Fabric Data Warehouse need as input from Synapse?
4. When is staying on ADF a reasonable decision?
5. What replaces the self-hosted integration runtime in Fabric?
6. Why use shortcuts during migration?

<details>
<summary>Answers</summary>

1. Fabric Data Warehouse.
2. Connections; datasets are removed, and their settings live in the activities.
3. A DACPAC exported from the dedicated SQL pool (or a connection to a supported source system).
4. When workloads are SSIS-heavy, not centred on Fabric analytics, or stable with no new requirements and no applicable retirement notice.
5. The on-premises data gateway (and the virtual network data gateway for VNet-hosted sources).
6. To let Fabric read existing ADLS data in place, enabling coexistence and gradual migration without copying data.

</details>

## References

- [Migrate from Azure Data Factory to Fabric Data Factory: planning](https://learn.microsoft.com/fabric/data-factory/migrate-planning-azure-data-factory)
- [Upgrade paths from ADF to Fabric](https://learn.microsoft.com/fabric/data-factory/upgrade-paths)
- [PowerShell upgrade module for ADF to Fabric](https://learn.microsoft.com/fabric/data-factory/migrate-pipelines-powershell-upgrade-module-for-azure-data-factory-to-fabric)
- [Assess your ADF to Fabric migration](https://learn.microsoft.com/azure/data-factory/how-to-assess-your-azure-data-factory-to-fabric-data-factory-migration)
- [Migration Assistant for Fabric Data Warehouse](https://learn.microsoft.com/fabric/data-warehouse/migration-assistant)
- [Migrate a Synapse dedicated SQL pool to Fabric Warehouse](https://learn.microsoft.com/fabric/data-warehouse/migration-synapse-dedicated-sql-pool-warehouse)
- [Migrate Synapse Spark to Fabric](https://learn.microsoft.com/fabric/data-engineering/migrate-synapse-overview)
- [What is Azure Synapse Analytics?](https://learn.microsoft.com/azure/synapse-analytics/overview-what-is)
