# Modern Data Architecture

> Topic: 12 · Data and Analytics · Tracks: E ○ A ● C ● · Status checked: 2026-11

## Why it matters

Every serious AI conversation turns into a data conversation. RAG needs clean, current, permissioned content; agents need reliable structured data; and fine-tuning needs curated examples. Over the last decade the industry moved from separate data warehouses and data lakes to the **lakehouse**, organised in **medallion layers**, stored in **open table formats** (Delta Lake, Apache Iceberg), and increasingly owned in a **data mesh** of domain teams. Architects and consultants need this vocabulary to assess a client's data estate and to design on Microsoft Fabric or Azure Databricks.

## Key concepts

### From warehouse and lake to lakehouse

| Architecture | Strengths | Weaknesses |
|---|---|---|
| **Data warehouse** | Fast SQL, strong schema, ACID, BI-friendly | Expensive storage, poor fit for unstructured data and ML; proprietary formats |
| **Data lake** | Cheap object storage for any format; great for ML and raw data | "Data swamp" risk; no transactions, weak schema enforcement, slow BI |
| **Lambda / two-tier** (lake + warehouse) | Combines both | Duplicate copies, pipelines and governance; staleness between tiers |
| **Lakehouse** | One copy of data on open formats in object storage, with ACID tables, schema enforcement, time travel, and both SQL and Spark engines | Requires good table maintenance and governance discipline |

A lakehouse is made possible by **open table formats**: a transaction log plus metadata over Parquet files that gives database-like guarantees on a lake.

### Open table formats

| | **Delta Lake** | **Apache Iceberg** | Apache Hudi |
|---|---|---|---|
| Origin | Databricks, now Linux Foundation | Netflix, now Apache Software Foundation | Uber, Apache |
| Metadata | `_delta_log` JSON commits + Parquet checkpoints | Metadata files, manifest lists, manifests | Timeline in `.hoodie` |
| Strengths | ACID, time travel, MERGE, schema evolution, deletion vectors, liquid clustering (Databricks) | Engine-neutral design, hidden partitioning, partition evolution, wide engine support (Snowflake, Trino, Spark, Athena) | Upsert-heavy, incremental processing |
| Microsoft platform | **Native format of Fabric OneLake** and Azure Databricks | Fabric reads Iceberg through **shortcuts** with metadata virtualisation (Delta ↔ Iceberg); Databricks supports Iceberg reads and managed Iceberg via Unity Catalog | Limited |

Interoperability matters more than picking a winner. **Delta UniForm** (Databricks) and **OneLake table format virtualisation** let one copy of data be read as both Delta and Iceberg. Check current support levels before you rely on them.

### Medallion architecture

A layered refinement pattern popularised by Databricks and used by default in Fabric guidance:

| Layer | Contents | Typical operations | Consumers |
|---|---|---|---|
| **Bronze (raw)** | Source data as landed, append-only, with ingestion metadata | Ingest, capture, minimal parsing | Data engineers, replay, audit |
| **Silver (cleansed, conformed)** | Deduplicated, validated, typed, joined, conformed keys; often SCD Type 2 history | Cleansing, quality checks, MERGE/upserts, PII handling | Data scientists, analysts, ML features, **RAG source content** |
| **Gold (curated, business-ready)** | Aggregates, star schemas, data products, feature tables | Business logic, dimensional modelling | Power BI, apps, agents, executives |

Medallion layers are a convention, not a law. Some teams add a landing zone before bronze or a platinum layer for specific consumers. What matters is clear contracts and quality gates between layers.

### Data mesh

**Data mesh** (Zhamak Dehghani) is an *organisational and operating model*, not a technology. It has four principles:

| Principle | Meaning | On Microsoft platforms |
|---|---|---|
| **Domain ownership** | Business domains own their data end to end | Fabric **domains** and workspaces per domain; Databricks catalogs per domain |
| **Data as a product** | Data is published with owners, SLAs, documentation, quality and discoverability | Purview Unified Catalog **data products**; endorsed and certified Fabric items |
| **Self-serve data platform** | A central platform team provides tools and paved roads | Fabric capacity and workspace templates; Databricks workspaces, Unity Catalog |
| **Federated computational governance** | Global standards enforced automatically, applied locally | Purview policies, sensitivity labels, Fabric tenant and domain settings, Unity Catalog grants |

Mesh suits large organisations with many domains and mature teams. Small organisations usually do better with a central lakehouse and clear ownership.

### Other patterns you will meet

| Pattern | Description |
|---|---|
| **Data fabric** | An architecture emphasising metadata-driven integration and virtualisation across sources (shortcuts, mirroring, catalogue) |
| **Kappa / streaming-first** | Treat everything as a stream; batch is a special case (Eventstreams, Event Hubs, Spark Structured Streaming) |
| **ELT over ETL** | Land raw data first, transform inside the lakehouse with SQL or Spark |
| **Zero-copy / virtualisation** | Access data in place (OneLake shortcuts, mirroring metadata, Delta Sharing) instead of copying |
| **Reverse ETL** | Push curated data back into operational systems (for example Cosmos DB in Fabric serving gold data to apps) |

## How it works on Azure

| Capability | Microsoft Fabric | Azure Databricks |
|---|---|---|
| Storage | **OneLake** (one logical lake per tenant, ADLS Gen2-compatible APIs) | Your ADLS Gen2 accounts, governed by Unity Catalog |
| Table format | Delta (Iceberg via virtualisation and shortcuts) | Delta (plus Iceberg support) |
| Engines | Spark, T-SQL (Warehouse and SQL analytics endpoint), KQL, Power BI (Direct Lake) | Spark, Databricks SQL, Photon |
| Ingestion | Pipelines, Dataflows Gen2, Copy jobs, Mirroring, Eventstreams, shortcuts | Lakeflow Connect, Auto Loader, Lakeflow Spark Declarative Pipelines, jobs |
| Governance | Domains, workspaces, OneLake security, Purview integration | Unity Catalog, plus Purview scanning |
| BI | Power BI native | Power BI connector, Databricks AI/BI |

Many enterprises use both. A common pattern is Databricks for heavy engineering and ML, with Fabric and Power BI for consumption, connected through **mirrored Azure Databricks catalogs** or shortcuts so data isn't copied.

### A reference flow

```text
Sources: SQL/Oracle/SAP, SaaS, files, events, IoT
   │  Mirroring / pipelines / Eventstreams / shortcuts
   ▼
Bronze (Delta, append-only, raw)  ──►  Silver (clean, conformed, PII-tagged)  ──►  Gold (star schemas, data products)
                                              │                                      │
                                              ├─► chunk + embed → AI Search (RAG)     ├─► Power BI (Direct Lake)
                                              └─► features → Azure ML / Foundry      └─► Fabric data agents, apps
Governance across all layers: Purview catalogue, lineage, sensitivity labels, quality rules
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Lakehouse vs warehouse-first | Lakehouse when you have mixed workloads (BI + ML + AI); warehouse-first (Fabric Warehouse) when teams are SQL-centric and data is mostly structured. In Fabric both sit on the same OneLake Delta storage |
| Delta vs Iceberg | On Microsoft platforms default to Delta; ensure Iceberg readability where other engines (Snowflake, Trino) must read the same data |
| Number of layers | Three is the norm; add a landing zone if raw files need staging. Don't create layers without consumers |
| Mesh vs central | Start central with domain-aligned workspaces; move to mesh principles as domain teams mature |
| Copy vs virtualise | Virtualise (shortcuts, mirroring) to avoid duplication and staleness; copy when you need transformation, isolation or performance |
| One platform vs two | Fabric alone for simplicity and Power BI-centric estates; Databricks + Fabric when heavy Spark engineering, MLOps or multicloud matter |

## Hands-on

1. Sketch the current data estate of a sample organisation (or your own) and classify each store as warehouse, lake or operational.
2. In a Fabric trial, create a lakehouse; upload a CSV to `Files/bronze`; use a notebook to write a cleansed Delta table `silver_customers` and an aggregated `gold_sales_by_region`.
3. Run `DESCRIBE HISTORY silver_customers` and query an older version with `VERSION AS OF` (time travel).
4. Write a one-page data mesh assessment: which domains exist, who would own which data product, and which governance rules must be global.

## Security, cost and reliability notes

- **Security:** classify and tag PII at silver at the latest; apply masking or separate secure tables before gold and before anything is exposed to AI.
- **Security:** least-privilege access per layer. Most users and agents should only see gold.
- **Cost:** open formats reduce lock-in, but uncontrolled copies across layers and platforms multiply storage and compute. Virtualise where you can.
- **Cost:** small files and unoptimised tables waste compute. Schedule OPTIMIZE/compaction and VACUUM (or rely on platform auto-optimisation).
- **Reliability:** keep bronze immutable so you can rebuild silver and gold. Use time travel for recovery, but understand that VACUUM limits how far back you can go.

## Common mistakes

- Calling a folder of CSVs a "lakehouse".
- Treating data mesh as a tool purchase instead of an operating model change.
- Building gold tables nobody consumes.
- Letting RAG pipelines read bronze data with unvetted PII.
- Aggressive VACUUM that breaks time travel and downstream readers.
- Choosing a table format without checking which engines must read it.

## Check yourself

1. What makes a lakehouse different from a data lake?
2. Describe the purpose of bronze, silver and gold layers.
3. Name the four principles of data mesh.
4. What does an open table format add on top of Parquet files?
5. When would you prefer virtualisation over copying data?
6. Which layer is usually the right source for RAG content, and why?

<details>
<summary>Answers</summary>

1. A lakehouse adds ACID transactions, schema enforcement, time travel and SQL performance on top of open files in the lake (through an open table format), so one copy serves BI and ML.
2. Bronze keeps raw data as landed for audit and replay; silver cleanses, deduplicates and conforms; gold provides business-ready aggregates and models for consumption.
3. Domain ownership, data as a product, self-serve data platform, federated computational governance.
4. A transaction log and metadata that provide ACID commits, versioning and time travel, schema enforcement and evolution, and efficient updates and deletes.
5. When copies would create staleness, duplication or extra governance burden, and the source can serve the access pattern adequately.
6. Silver (or a curated gold content set), because it is cleansed and deduplicated and has PII handled, while still detailed enough for retrieval.

</details>

## References

- [What is a lakehouse in Microsoft Fabric?](https://learn.microsoft.com/fabric/data-engineering/lakehouse-overview)
- [Implement medallion lakehouse architecture in Fabric](https://learn.microsoft.com/fabric/onelake/onelake-medallion-lakehouse-architecture)
- [What is the medallion lakehouse architecture? (Azure Databricks)](https://learn.microsoft.com/azure/databricks/lakehouse/medallion)
- [Data mesh (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/architectures/what-is-data-mesh)
- [Use Iceberg tables with OneLake](https://learn.microsoft.com/fabric/onelake/onelake-iceberg-tables)
- [What is Delta Lake? (Azure Databricks)](https://learn.microsoft.com/azure/databricks/delta/)
- [Cloud-scale analytics (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/)
