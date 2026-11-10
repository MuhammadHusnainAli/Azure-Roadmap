# Microsoft Fabric Fundamentals

## Why it matters

Microsoft Fabric is Microsoft's software-as-a-service analytics platform. Data integration, engineering, warehousing, real-time analytics, data science, Power BI, databases and AI agents all share one storage layer (**OneLake**), one capacity-based billing model and one governance model. Most new Microsoft data and AI engagements involve Fabric, and Fabric data is increasingly a knowledge source for Foundry agents. Consultants get asked about licensing and capacity sizing in nearly every Fabric conversation, and those topics are easy to get wrong.

## Key concepts

### The building blocks

| Concept | Meaning |
|---|---|
| **Tenant** | Your Microsoft Entra tenant; Fabric is enabled per tenant by a Fabric administrator |
| **Capacity** | A pool of compute (measured in **capacity units, CUs**) that runs all Fabric workloads for the workspaces assigned to it |
| **Workspace** | A container for items (lakehouses, warehouses, notebooks, reports...) assigned to a capacity; the main unit of collaboration and access |
| **Item** | A Fabric artefact: lakehouse, warehouse, notebook, pipeline, eventhouse, semantic model, report, data agent, SQL database... |
| **Domain** | A logical grouping of workspaces by business area (for data mesh-style governance and discovery) |
| **OneLake** | One logical data lake per tenant, automatically provisioned, where all Fabric data is stored in open formats (Delta Parquet) |
| **Workloads / experiences** | Data Factory, Data Engineering, Data Warehouse, Data Science, Real-Time Intelligence, Databases, Power BI, Fabric IQ, Industry solutions |

### OneLake

- **One per tenant.** Like OneDrive for data, every workspace and item stores its data in the same logical lake.
- **Open format.** Tables are Delta Parquet. Any engine (Spark, T-SQL, KQL, Analysis Services) reads the same copy.
- **ADLS Gen2-compatible APIs.** Tools that speak ADLS/Blob APIs can read and write at `https://onelake.dfs.fabric.microsoft.com/<workspace>/<item>/...`.
- **OneLake catalog.** The central place to discover, explore and govern items across workspaces.
- **OneLake security.** Role-based access to tables and folders inside items (check current status of fine-grained features in your tenant).

### Shortcuts: zero-copy access

A **shortcut** is a pointer inside a lakehouse or KQL database to data stored elsewhere. It works like a symbolic link: no data is copied.

| Shortcut targets (checked 2026-11) |
|---|
| Internal OneLake locations: lakehouses, warehouses, KQL databases, mirrored databases, mirrored Databricks catalogs, SQL databases, semantic models |
| Azure Data Lake Storage Gen2, Azure Blob Storage |
| Amazon S3 and S3-compatible storage, Google Cloud Storage |
| Dataverse |
| OneDrive and SharePoint |
| Iceberg tables (exposed as Delta through metadata virtualisation) |
| On-premises or network-restricted storage through the on-premises data gateway |

Key behaviours:

- Shortcuts in the **Tables** folder must be at the top level and point to Delta (or Iceberg) tables. Shortcuts in **Files** can go anywhere and point to any format.
- Internal shortcuts authorise with the **calling user's** identity. External shortcuts use a **cloud connection** (delegated credentials).
- **Shortcut caching** (1–28 days) reduces cross-cloud egress for GCS, S3, S3-compatible and gateway shortcuts.
- Deleting a shortcut doesn't delete the target, but deleting files *inside* a shortcut does delete them at the target if you have permission.

### Mirroring: near real-time replicas

**Mirroring** continuously replicates operational databases into OneLake as Delta tables, with a SQL analytics endpoint, and no ETL pipelines to build.

| Type | How | Examples (checked 2026-11) |
|---|---|---|
| **Database mirroring** | Replicates data and changes into OneLake | Azure SQL Database, Azure SQL Managed Instance, SQL Server, Azure Cosmos DB, Azure Database for PostgreSQL, Azure Database for MySQL (preview), Snowflake, Google BigQuery, Oracle, SAP, SharePoint lists (preview), Fabric SQL database (automatic) |
| **Metadata mirroring** | Syncs catalogue metadata; data stays in place via shortcuts | Azure Databricks Unity Catalog, Snowflake, Dremio (preview) |
| **Open mirroring** | Any application writes change data to a landing zone using public APIs | Partner and custom sources |

Cost: replication compute is **free** and doesn't consume capacity. Mirrored storage is free up to **1 TB per CU** purchased (an F64 includes 64 TB of mirroring storage). Querying mirrored data with SQL, Spark or Power BI consumes capacity normally. A running capacity is required; if it's paused, replication stops.

### Capacities and SKUs

| SKU | CUs | Power BI equivalent (compute comparison) |
|---|---|---|
| F2 | 2 | — |
| F4 | 4 | — |
| F8 | 8 | EM1 / A1 |
| F16 | 16 | EM2 / A2 |
| F32 | 32 | EM3 / A3 |
| **F64** | 64 | **P1** / A4 |
| F128 – F2048 | 128 – 2,048 | P2 – P5 (up to F1024) |
| F4096, F8192 | 4,096, 8,192 | — |
| Trial | 64 | — |

- **F SKUs** are bought in Azure: billed per second (one-minute minimum), no commitment, can be **paused and resumed**, with optional one-year **reservations** for a discount. Microsoft recommends F SKUs.
- **P SKUs** (Power BI Premium per capacity) support Fabric when enabled, but they are being retired for new purchases. Plan on F SKUs.
- **A and EM SKUs** support only Power BI items, not Fabric workloads.
- The **trial** gives a 64-CU capacity for **60 days**.

### Smoothing, bursting and throttling

Fabric lets operations **burst** above the capacity's size and then **smooths** the usage over time:

| Operation type | Smoothed over | Examples |
|---|---|---|
| **Interactive** | 5 to 64 minutes, depending on CU usage | Report queries, SQL queries from users |
| **Background** | 24 hours | Pipelines, Spark jobs, semantic model refresh, most Warehouse operations |

Usage is tracked in 30-second **timepoints**. When smoothed usage consumes future capacity, throttling escalates:

| Future capacity already consumed | Stage | Effect |
|---|---|---|
| ≤ 10 minutes | Overage protection | No throttling |
| 10–60 minutes | Interactive delay | New interactive operations delayed 20 seconds |
| 60 minutes – 24 hours | Interactive rejection | New interactive operations rejected; background still runs |
| > 24 hours | Background rejection | All new requests rejected until the carry-forward burns down |

In-flight operations always finish. Real-Time Intelligence skips the 20-second delay stage. To recover faster on F SKUs, scale up temporarily, pause and resume (which bills the accumulated usage), or enable **capacity overage** billing (charged at three times the normal rate). Use the **Fabric Capacity Metrics app** to see utilisation, throttling and burndown. Note that **Autoscale Billing for Spark** moves Spark to pay-as-you-go outside smoothing.

### Per-user licences

| Licence | What it allows |
|---|---|
| **Fabric (Free)** | Assigned on first sign-in. Create and share **non-Power BI** Fabric items in workspaces on F or Trial capacity. On **F64 or larger** (or P capacity), view Power BI content with a Viewer role |
| **Power BI Pro** | Create, share and collaborate on Power BI content. Needed to view Power BI content on capacities **smaller than F64** |
| **Premium Per User (PPU)** | Most Power BI Premium features per user. **Doesn't** provide a Fabric capacity, so you can't create lakehouses, notebooks and so on with PPU alone |

The **F64 line** is the most important licensing fact: below F64, every Power BI viewer needs Pro or PPU; at F64 and above, Free users with the Viewer role can view.

### Workspace roles

| Role | Can do |
|---|---|
| **Admin** | Everything, including managing access and deleting the workspace |
| **Member** | Add others (except admins), share, publish and edit all content |
| **Contributor** | Create, edit and delete content; can't manage access |
| **Viewer** | View and read content (and query SQL endpoints) without editing |

Item-level sharing and OneLake security let you grant narrower access than workspace roles.

## How it works on Azure

### Create and manage an F capacity

```bash
# Fabric capacities are Azure resources: Microsoft.Fabric/capacities
az fabric capacity create -g rg-fabric --capacity-name fabcontosoprod -l westeurope \
  --sku "{name:F64,tier:Fabric}" \
  --administration "{members:[admin1@contoso.com,admin2@contoso.com]}"

az fabric capacity suspend -g rg-fabric --capacity-name fabcontosoprod   # pause: stop compute billing
az fabric capacity resume  -g rg-fabric --capacity-name fabcontosoprod
az fabric capacity update  -g rg-fabric --capacity-name fabcontosoprod --sku "{name:F128,tier:Fabric}"  # scale
```

The `az fabric` commands come from the `microsoft-fabric` CLI extension. Bicep and Terraform (`azurerm_fabric_capacity`) are also available. Workspaces and items are managed through the **Fabric REST APIs**, the **Fabric CLI** (`fab`) or the **Fabric Terraform provider** (`microsoft/fabric`).

### Administration essentials

| Area | Where |
|---|---|
| Tenant settings (enable Fabric, Copilot, data agents, external sharing...) | Admin portal (Fabric administrator) |
| Capacity settings, assignment, delegated settings | Admin portal / Azure portal |
| Domains and domain admins | Admin portal |
| Monitoring | Capacity Metrics app, Monitoring hub, workspace monitoring |
| Governance | Purview hub in Fabric, sensitivity labels, endorsement (promoted, certified) |
| ALM | Git integration (Azure DevOps, GitHub), deployment pipelines, variable libraries |
| Networking | Private links (tenant and workspace level), managed private endpoints, outbound access protection, trusted workspace access to storage |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Capacity size | Start from a proof of concept with the Metrics app; F64 is the practical minimum when many Free-licensed users must view Power BI content |
| One vs many capacities | Separate capacities for production vs development, for noisy workloads (heavy Spark) vs interactive BI, or for chargeback and regions |
| Workspace design | One workspace per domain × environment (dev/test/prod) per data product is a common pattern; avoid giant shared workspaces |
| Copy vs shortcut vs mirror | Shortcut when data is already in an open format elsewhere; mirror for operational databases; copy (pipelines) when you need transformation at ingestion |
| Pay-as-you-go vs reservation | Reserve the steady baseline; use pay-as-you-go capacities you can pause for dev/test or burst projects |
| Region | Capacity region determines where data at rest lives; align with data residency requirements and with Foundry and AI resources |

## Hands-on

1. Start a Fabric trial (or create an F2 in Azure) and create a workspace assigned to it.
2. Create a lakehouse and add a shortcut to an ADLS Gen2 container (or Amazon S3 bucket) containing Parquet or Delta data.
3. Mirror an Azure SQL Database (or a sample database) and query the mirrored tables through the SQL analytics endpoint.
4. Install the **Fabric Capacity Metrics** app and identify interactive vs background CU usage.
5. Pause and resume the F capacity from the CLI and observe what stops working while it's paused.

## Security, cost and reliability notes

- **Security:** treat tenant settings as policy: restrict who can create workspaces, share externally, and use Copilot and data agents.
- **Security:** use private links and outbound access protection for sensitive tenants or workspaces; use workspace identity and trusted workspace access to reach firewalled storage.
- **Security:** shortcuts and mirroring expose data in new places. Review who can read the target item, because internal shortcuts use the caller's identity.
- **Cost:** a paused capacity stops compute billing but OneLake storage is still billed, and mirroring stops. Items on a paused capacity are unavailable.
- **Cost:** heavy background jobs eat into the smoothed 24-hour budget. Schedule them, right-size Spark, and isolate them on separate capacities when needed.
- **Reliability:** OneLake supports zone-redundant storage in supported regions and has BCDR settings per capacity (enable disaster recovery capacity settings for cross-region replication of OneLake data where needed).

## Common mistakes

- Assuming PPU gives access to lakehouses and notebooks.
- Buying F32 for an organisation of thousands of report viewers, then discovering every viewer needs Pro.
- One workspace for everything, so access control and ALM become impossible.
- Pausing a production capacity "to save money" and breaking mirroring and reports.
- Copying data from ADLS into OneLake when a shortcut would do.
- Not monitoring throttling until users complain.

## Check yourself

1. What is OneLake and how many exist per tenant?
2. What is the difference between a shortcut and mirroring?
3. Why is F64 an important threshold?
4. Can a Premium Per User licence alone run a Spark notebook in Fabric?
5. What happens to mirroring when the capacity is paused?
6. How does Fabric smooth background operations?
7. Name the four workspace roles.

<details>
<summary>Answers</summary>

1. A single logical, tenant-wide data lake, automatically provisioned, storing all Fabric data in open Delta Parquet format with ADLS-compatible APIs. One per tenant.
2. A shortcut references data in place with no copy. Mirroring continuously replicates a database (or for metadata mirroring, its catalogue) into OneLake as Delta tables.
3. At F64 and above, users with a Free licence and a Viewer role can view Power BI content; below F64, viewers need Pro, PPU or a trial.
4. No. PPU doesn't provide a Fabric capacity; you need an F (or Trial or enabled P) capacity.
5. Replication stops; mirrored storage beyond the free limit, and while paused, is billed as OneLake storage.
6. Background operations are smoothed over 24 hours (interactive over 5 to 64 minutes), so short bursts above capacity are allowed; throttling starts only when more than 10 minutes of future capacity is consumed.
7. Admin, Member, Contributor, Viewer.

</details>

## References

- [What is Microsoft Fabric?](https://learn.microsoft.com/fabric/fundamentals/microsoft-fabric-overview)
- [Fabric licenses and capacity](https://learn.microsoft.com/fabric/enterprise/licenses)
- [OneLake overview](https://learn.microsoft.com/fabric/onelake/onelake-overview)
- [OneLake shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-shortcuts)
- [Mirroring in Fabric](https://learn.microsoft.com/fabric/mirroring/overview)
- [Throttling in Fabric](https://learn.microsoft.com/fabric/enterprise/throttling)
- [Fabric Capacity Metrics app](https://learn.microsoft.com/fabric/enterprise/metrics-app)
- [Workspace roles](https://learn.microsoft.com/fabric/fundamentals/roles-workspaces)
- [Fabric domains](https://learn.microsoft.com/fabric/governance/domains)
