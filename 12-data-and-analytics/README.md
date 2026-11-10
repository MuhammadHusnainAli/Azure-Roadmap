# 12 · Data and Analytics

> Tracks: E ○ A ● C ● · Part C · Data and Analytics

AI is built on data, and on Azure that increasingly means **Microsoft Fabric**: one SaaS platform with OneLake, engineering, warehousing, operational databases, real-time intelligence, Power BI and data agents. This topic starts with the architecture vocabulary (lakehouse, medallion, mesh, open table formats), works through each Fabric workload, and covers **Azure Databricks**, the existing **Data Factory and Synapse** estates and how to migrate them, **Azure Data Explorer and Stream Analytics** for telemetry, and **Microsoft Purview** for governance. It ends with what makes data ready for RAG, agents and fine-tuning.

## What you will be able to do

- Explain lakehouse, medallion, data mesh, Delta and Iceberg, and choose an architecture for a client.
- Size and license Fabric correctly (capacities, the F64 line, Pro vs PPU vs Free) and explain smoothing and throttling.
- Use OneLake shortcuts and mirroring to avoid copying data.
- Build medallion pipelines with lakehouses, Spark notebooks, pipelines, Copy jobs and Dataflows Gen2.
- Choose between Warehouse, lakehouse, SQL database in Fabric and Cosmos DB in Fabric.
- Ingest and analyse streams with Eventstreams, Eventhouse, KQL and Activator.
- Model Power BI semantic models with Direct Lake and DAX, and prepare them for Copilot.
- Build Fabric data agents and connect them to Foundry agents with user identity.
- Position Azure Databricks (Unity Catalog, Lakeflow, Mosaic AI) alongside Fabric and Foundry.
- Plan an ADF and Synapse migration to Fabric with Microsoft's migration tools.
- Govern data with Purview Unified Catalog: domains, data products, quality and lineage.
- Assess AI data readiness: quality, freshness, permissions, semantics and compliance.

## Prerequisites

[Topic 08](../08-storage/README.md) (ADLS Gen2), [Topic 09](../09-databases/README.md) (SQL and Cosmos DB), [Topic 11](../11-integration-messaging-and-apis/README.md) (Event Hubs, Data Factory). Basic SQL is assumed.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Modern Data Architecture](01-modern-data-architecture.md) | 45 min | Fabric notebook |
| 02 | [Microsoft Fabric Fundamentals](02-microsoft-fabric-fundamentals.md) | 60 min | Portal, CLI |
| 03 | [Fabric Data Engineering and Data Factory](03-fabric-data-engineering-and-data-factory.md) | 60 min | PySpark, pipelines |
| 04 | [Fabric Data Warehouse and Databases](04-fabric-data-warehouse-and-databases.md) | 50 min | T-SQL, Python |
| 05 | [Fabric Real-Time Intelligence](05-fabric-real-time-intelligence.md) | 50 min | KQL, Eventstream |
| 06 | [Power BI and Semantic Models](06-power-bi-and-semantic-models.md) | 60 min | Power BI, DAX |
| 07 | [Fabric IQ and Data Agents](07-fabric-iq-and-data-agents.md) | 50 min | Fabric, Foundry |
| 08 | [Azure Databricks](08-azure-databricks.md) | 55 min | SQL, PySpark, Bicep |
| 09 | [Azure Data Factory and Azure Synapse Analytics](09-azure-data-factory-and-azure-synapse-analytics.md) | 45 min | Migration tools |
| 10 | [Azure Data Explorer and Stream Analytics](10-azure-data-explorer-and-stream-analytics.md) | 45 min | KQL, ASA SQL |
| 11 | [Microsoft Purview Data Governance](11-microsoft-purview-data-governance.md) | 45 min | Purview portal |
| 12 | [Data Platform for AI Readiness](12-data-platform-for-ai-readiness.md) | 45 min | Assessment |

**AI engineers:** 01, 02, 07 and 12 in depth; 03 and 05 for working knowledge. **Architects:** all of it, with 01, 02, 07, 08 and 12 in depth. **Consultants:** 01, 02, 06, 07, 09, 11 and 12.

## Completion checklist

- [ ] I can explain the medallion architecture and when data mesh is appropriate.
- [ ] I can explain the F64 licensing threshold and why PPU doesn't give access to Fabric workloads.
- [ ] I have created a shortcut and a mirrored database and queried both.
- [ ] I have built a bronze → silver → gold notebook with MERGE and run it from a pipeline.
- [ ] I can choose between Warehouse, lakehouse, SQL database in Fabric and Cosmos DB in Fabric.
- [ ] I have written KQL with `summarize ... by bin()` and created an Activator rule.
- [ ] I have built a Direct Lake semantic model with explicit DAX measures and RLS.
- [ ] I have created a Fabric data agent and called it from a Foundry agent.
- [ ] I can describe Unity Catalog's namespace and how Databricks data appears in Fabric.
- [ ] I can outline an ADF/Synapse to Fabric migration plan with the right tools.
- [ ] I have published a data product with quality rules in Purview.
- [ ] I can run an AI data readiness assessment for a use case.

## Maps to

- **DP-700** (Fabric Data Engineer Associate): ingest, transform, Real-Time Intelligence, lakehouse, warehouse, monitoring.
- **DP-600** (Fabric Analytics Engineer Associate): semantic models, Direct Lake, DAX (confirm current status on Microsoft Learn).
- **AZ-305**: design data storage, data integration and analytics solutions.
- **AB-100**: data readiness and data agents in agentic business solutions.

## Next

[13 · AI and Generative AI Foundations](../13-ai-and-generative-ai-foundations/README.md)
