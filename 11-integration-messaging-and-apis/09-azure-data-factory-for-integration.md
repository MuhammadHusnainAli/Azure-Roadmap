# Azure Data Factory for Integration

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ○ A ● C ○ · Status checked: 2026-11

## Why it matters

Not every integration is a message or an API call. A lot of enterprise integration is **moving data in bulk on a schedule**: nightly extracts from SAP or Oracle, file drops from partners over SFTP, copying on-premises SQL tables into a data lake, or loading documents into a RAG pipeline. **Azure Data Factory (ADF)** is Azure's managed service for this kind of data integration, with 90+ connectors, a hybrid runtime for on-premises sources, and orchestration with retries and monitoring. This file covers ADF as an *integration* tool. Topic 12 covers it as part of the data platform, along with Fabric Data Factory and migration.

## Key concepts

### Building blocks

| Concept | Meaning |
|---|---|
| **Pipeline** | A logical group of activities that performs a unit of work |
| **Activity** | A step: **data movement** (Copy), **data transformation** (Data Flow, Databricks notebook, stored procedure, Azure Function...), or **control** (ForEach, If, Until, Lookup, Web, Wait, Execute Pipeline) |
| **Dataset** | A named view of data in a store (a table, a folder of files) |
| **Linked service** | Connection information for a data store or compute (like a connection string, ideally using managed identity) |
| **Integration runtime (IR)** | The compute that runs activities |
| **Trigger** | Starts pipeline runs: schedule, tumbling window, storage event, custom event (Event Grid) |
| **Parameters and variables** | Make pipelines reusable (for example one pipeline for many tables) |
| **Mapping data flow** | Visually designed transformations executed on managed Spark clusters |

### Integration runtimes

| IR type | Runs where | Use for |
|---|---|---|
| **Azure IR** | Microsoft-managed compute in Azure; can run in a **managed virtual network** with managed private endpoints | Cloud-to-cloud copy, data flows, dispatching activities |
| **Self-hosted IR (SHIR)** | Software you install on Windows machines on-premises or in a VNet | Reaching on-premises or privately networked sources (SQL Server, Oracle, SAP, file shares) |
| **Azure-SSIS IR** | Managed cluster that runs SQL Server Integration Services packages | Lift-and-shift of existing SSIS packages |

The SHIR makes **outbound** HTTPS connections only (to the ADF service and Service Bus relay endpoints), so you don't open inbound firewall ports. Install at least two nodes for high availability.

### Connectors

ADF supports 90+ connectors across Azure data stores, databases (SQL Server, Oracle, DB2, MySQL, PostgreSQL, SAP HANA and others), SaaS (Salesforce, Dynamics 365, ServiceNow and others), file systems (SFTP, FTP, HDFS, file share), generic protocols (REST, HTTP, OData, ODBC) and SAP (SAP Table, SAP BW, SAP CDC). Each connector's documentation lists which IR types and activities it supports. Some legacy connector versions are being retired in favour of new versions, so check the connector page for deprecation notices.

## How it works on Azure

### A typical integration pipeline

```text
Trigger: tumbling window (hourly, with dependency on previous window)
  └─ Lookup: get list of tables changed (watermark table)
  └─ ForEach table (parallel, batch count 8)
        └─ Copy: on-prem SQL (via SHIR) → ADLS Gen2 /raw/<table>/<window>/ as Parquet
        └─ Stored procedure: update watermark
  └─ On failure: Web activity → Logic App → Teams alert
```

### Copy activity essentials

| Setting | Why it matters |
|---|---|
| **Incremental loading** | Watermark columns, change tracking, or CDC to avoid full reloads |
| **Data integration units (DIUs)** | Scale Azure IR copy throughput; more DIUs cost more |
| **Parallel copies** | Partitioned reads from sources that support them |
| **Fault tolerance** | Skip incompatible rows and log them instead of failing the whole run |
| **Staged copy** | Stage through Blob storage, for example for PolyBase/COPY into Synapse or across networks |
| **Data consistency verification** | Verify file checksums after binary copy |

### ADF with event-driven integration

| Pattern | How |
|---|---|
| File arrives → process | **Storage event trigger** (BlobCreated via Event Grid) starts the pipeline |
| Business event → load | **Custom event trigger** on an Event Grid custom topic |
| Pipeline finished → downstream | Web activity calls a Logic App or Function; or publish to Service Bus / Event Grid |
| API ingestion | REST connector with pagination rules into the lake |

### Feeding RAG and AI pipelines

ADF is a sensible tool for **bulk, scheduled** movement of source content into a landing zone (Blob or ADLS) before AI-specific processing:

```text
SharePoint / file shares / databases ──ADF copy (scheduled, incremental)──► ADLS /raw
ADLS BlobCreated ──Event Grid──► Functions/Container Apps: extract, chunk, embed ──► Azure AI Search
```

Azure AI Search **indexers** can also pull directly from Blob, ADLS, SQL and Cosmos DB, so ADF isn't always needed (topic 17).

### Create with keyless linked services (CLI)

```bash
az datafactory create -g rg-int -n adf-contoso-prod -l westeurope
# Confirm the factory has a system-assigned managed identity (portal and PowerShell create one automatically;
# check the "identity" block in the output, and add one if it is missing)
az datafactory show -g rg-int -n adf-contoso-prod --query identity
```

Linked service JSON using the factory's managed identity to reach ADLS Gen2:

```json
{
  "name": "ls_adls_raw",
  "properties": {
    "type": "AzureBlobFS",
    "typeProperties": { "url": "https://stcontosoraw.dfs.core.windows.net" }
  }
}
```

With no credential specified, ADLS Gen2 uses the factory's **system-assigned managed identity**. Grant it **Storage Blob Data Contributor** on the account. For other secrets, use **Azure Key Vault linked services** rather than inline credentials.

### Networking

| Option | Detail |
|---|---|
| **Managed virtual network** | Azure IR runs in an ADF-managed VNet; connect to data stores using **managed private endpoints** that you approve on the target |
| **Private endpoint for ADF** | Private access to the ADF portal and/or the data plane used by SHIR |
| **SHIR in your VNet** | Reach private resources and on-premises (via VPN or ExpressRoute) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| ADF vs Logic Apps | ADF for **bulk data movement** and transformation at volume; Logic Apps for **event and message-level** integration with SaaS actions and approvals |
| ADF vs Fabric Data Factory | For new analytics platforms on Fabric, use Fabric Data Factory pipelines. For existing ADF estates, hybrid SHIR-heavy integration or non-Fabric targets, ADF remains fully supported (see topic 12 for migration) |
| Copy vs Mapping Data Flow vs Databricks | Copy for movement; Mapping Data Flows for visual transformations at moderate scale; Databricks/Spark for complex, large-scale transformation |
| Azure IR vs SHIR | Azure IR (with managed VNet) whenever the source is reachable from Azure; SHIR only when it isn't |
| Full vs incremental loads | Incremental by default; full loads only for small reference data |

## Hands-on

1. Create a data factory and an ADLS Gen2 account; grant the factory's managed identity **Storage Blob Data Contributor**.
2. Create a pipeline with a Copy activity from a public HTTP CSV into ADLS `/raw` as Parquet. Run it and inspect the monitoring view.
3. Parameterise the pipeline (source URL, target folder) and run it in a ForEach over three files.
4. Add a **storage event trigger** that starts a second pipeline when a file lands in `/incoming`.
5. Optional: install a self-hosted IR on a VM, register it, and copy from a SQL Server on that VM.

## Security, cost and reliability notes

- **Security:** use managed identity for linked services and Key Vault for any remaining secrets. Never put passwords in pipeline JSON committed to Git.
- **Security:** use a managed VNet with managed private endpoints, and disable public network access on data stores.
- **Security:** keep SHIR machines patched and dedicated; they hold credentials to source systems.
- **Cost:** you pay for orchestration (activity runs), data movement (DIU-hours), data flow cluster time (vCore-hours) and SHIR activity time. Over-parallelised ForEach loops and oversized data flow clusters are the usual cost surprises.
- **Reliability:** set activity retries and timeouts. Use tumbling window triggers with dependencies for reliable backfills. Make loads idempotent (overwrite partitions rather than append blindly).
- **Operations:** connect ADF to Git (Azure Repos or GitHub) and deploy through CI/CD with ARM templates or the npm-based build.

## Common mistakes

- Using ADF for per-message, low-latency integration.
- Full reloads of large tables every night.
- A single SHIR node, which becomes a single point of failure.
- Hard-coded connection strings instead of managed identity and Key Vault.
- Building pipelines only in the portal without Git integration.
- Ignoring connector deprecation notices until pipelines break.

## Check yourself

1. Name the three integration runtime types and when to use each.
2. What is the difference between a dataset and a linked service?
3. Which trigger type supports dependencies and backfill of time windows?
4. How does the self-hosted IR connect without inbound firewall rules?
5. When would you choose Logic Apps instead of ADF?
6. How do you let ADF reach a storage account that has public access disabled, without a SHIR?

<details>
<summary>Answers</summary>

1. Azure IR for cloud-reachable sources and data flows; self-hosted IR for on-premises or private network sources; Azure-SSIS IR to run existing SSIS packages.
2. A linked service is the connection to a store (like a connection string); a dataset is a named reference to specific data within it (a table or folder).
3. The tumbling window trigger.
4. It makes outbound HTTPS connections to the Data Factory service, so no inbound ports are needed.
5. For event- or message-level integration, SaaS actions, approvals and B2B, where per-item workflow logic matters more than bulk movement.
6. Use the Azure IR in a managed virtual network with a managed private endpoint to the storage account, approved on the storage side.

</details>

## References

- [What is Azure Data Factory?](https://learn.microsoft.com/azure/data-factory/introduction)
- [Integration runtime concepts](https://learn.microsoft.com/azure/data-factory/concepts-integration-runtime)
- [Connector overview](https://learn.microsoft.com/azure/data-factory/connector-overview)
- [Copy activity performance and scalability](https://learn.microsoft.com/azure/data-factory/copy-activity-performance)
- [Pipeline triggers](https://learn.microsoft.com/azure/data-factory/concepts-pipeline-execution-triggers)
- [Managed virtual network and managed private endpoints](https://learn.microsoft.com/azure/data-factory/managed-virtual-network-private-endpoint)
- [Managed identity for Data Factory](https://learn.microsoft.com/azure/data-factory/data-factory-service-identity)
- [Continuous integration and delivery](https://learn.microsoft.com/azure/data-factory/continuous-integration-delivery)
