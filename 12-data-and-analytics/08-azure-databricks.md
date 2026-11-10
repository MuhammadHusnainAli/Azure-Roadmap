# Azure Databricks

> Topic: 12 · Data and Analytics · Tracks: E ○ A ● C ○ · Status checked: 2026-11

## Why it matters

Azure Databricks is a first-party Azure service, jointly developed by Microsoft and Databricks, built on Apache Spark and Delta Lake. Many enterprises run their most demanding data engineering, data science and ML workloads on it, and it has strong governance (**Unity Catalog**), declarative pipelines (**Lakeflow**) and a full AI stack (**Mosaic AI**). Architects regularly design estates where Databricks and Fabric coexist, and AI engineers need to connect Databricks data, models and Genie spaces to Foundry agents.

## Key concepts

### Architecture

| Plane | Contains | Where it runs |
|---|---|---|
| **Control plane** | Web app, notebooks metadata, job scheduler, Unity Catalog services, cluster manager | Databricks-managed Azure subscription |
| **Classic compute plane** | Your clusters (VMs) for classic compute | **Your** Azure subscription, optionally in **your VNet** (VNet injection) |
| **Serverless compute plane** | Serverless notebooks, jobs, pipelines, SQL warehouses, model serving | Databricks-managed compute in your region, with network policies and private connectivity options |

A **workspace** is the environment teams work in (Premium tier for Unity Catalog and most enterprise features; the Standard tier is being retired, so check the current state). Workspaces attach to a regional **Unity Catalog metastore**.

### Compute options

| Type | Use |
|---|---|
| **Serverless compute** (notebooks, jobs, pipelines) | Fast start, no cluster management, billed per DBU; recommended default where available |
| **All-purpose (classic) clusters** | Interactive development with custom configuration, init scripts, GPU VMs |
| **Job clusters** | Ephemeral clusters for scheduled jobs (cheaper DBU rate than all-purpose) |
| **SQL warehouses** (Serverless, Pro, Classic) | Databricks SQL for BI and SQL analytics, with Photon |
| **Photon** | Vectorised C++ query engine accelerating SQL and DataFrame workloads |

Billing has two parts: **DBUs** (Databricks Units, by workload type and tier) plus, for classic compute, the **Azure VM** cost in your subscription. Serverless DBU prices include the compute.

### Unity Catalog

Unity Catalog is the unified governance layer for data and AI assets:

```text
Metastore (one per region per account)
 └─ Catalog (e.g. prod_sales)
     └─ Schema (e.g. silver)
         ├─ Tables (managed or external), views, materialized views
         ├─ Volumes (governed non-tabular files)
         ├─ Functions (SQL / Python UDFs, also usable as agent tools)
         └─ Models (registered ML models and versions)
```

| Feature | Description |
|---|---|
| **Three-level namespace** | `catalog.schema.object` |
| **Managed vs external tables** | Managed: Unity Catalog manages files and lifecycle (with predictive optimisation). External: you manage files in your own storage path |
| **Storage credentials and external locations** | Govern access to ADLS Gen2 through an **Azure Databricks access connector** (a managed identity), with no storage keys |
| **Privileges** | `GRANT SELECT ON TABLE ... TO group`, inherited down the hierarchy; row filters and column masks; attribute-based policies with tags (check status) |
| **Lineage** | Automatic table and column lineage across notebooks, jobs and dashboards |
| **Delta Sharing** | Open protocol to share data with other Databricks or non-Databricks recipients without copying |
| **System tables** | Audit logs, billing usage, lineage and more, queryable with SQL |
| **Federation** | Lakehouse Federation (query external databases) and catalog federation |

### Lakeflow

**Lakeflow** is Databricks' data engineering suite:

| Component | Formerly | Purpose |
|---|---|---|
| **Lakeflow Connect** | — | Managed ingestion connectors (SaaS apps such as Salesforce and Workday; databases with CDC) |
| **Lakeflow Spark Declarative Pipelines** | **Delta Live Tables (DLT)**, briefly "Lakeflow Declarative Pipelines" | Declarative pipelines of streaming tables and materialized views, with data quality **expectations**, automatic dependency handling and incremental processing |
| **Lakeflow Jobs** | Databricks Workflows | Orchestration of notebooks, pipelines, SQL, dbt, ML tasks |

Existing DLT pipelines continue to work. The Python API moved from `import dlt` to `from pyspark import pipelines as dp`, while older code remains supported. Check the "What happened to DLT" page for details.

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

@dp.table(comment="Raw orders from cloud storage via Auto Loader")
def bronze_orders():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load("/Volumes/prod/landing/orders/"))

@dp.table(comment="Cleaned orders")
@dp.expect_or_drop("valid_amount", "amount >= 0")
@dp.expect("has_customer", "customer_id IS NOT NULL")
def silver_orders():
    return (spark.readStream.table("bronze_orders")
            .withColumn("order_date", F.to_date("order_date")))
```

### Mosaic AI

| Capability | What it does |
|---|---|
| **Model Serving** | Serve custom models, open models and external models (including Azure OpenAI and other providers) behind one endpoint style; GPU serving; provisioned throughput for foundation models |
| **AI Gateway (Databricks)** | Usage tracking, rate limits, guardrails and payload logging on serving endpoints |
| **Vector Search** | Managed vector indexes synced from Delta tables (Delta Sync) |
| **Agent Framework and evaluation** | Build, evaluate (MLflow) and deploy agents; Unity Catalog functions as tools |
| **MLflow** | Experiment tracking, model registry in Unity Catalog, GenAI tracing and evaluation |
| **AI Functions** | SQL functions such as `ai_query`, `ai_classify`, `ai_extract` to call models from SQL |
| **AI/BI Genie** | Natural-language Q&A spaces over Unity Catalog data (comparable to Fabric data agents) |

Product names in this area change often (for example Agent Bricks), so check the Azure Databricks release notes.

### Integration with Microsoft Foundry and Fabric

| Integration | How |
|---|---|
| **Foundry agents → Databricks Genie** | Foundry Agent Service can connect to an Azure Databricks Genie space as a tool or connection, so agents can ask governed questions of Unity Catalog data (check current tool name and status) |
| **Databricks → Foundry Models** | Register Azure OpenAI / Foundry model deployments as **external models** in Model Serving, or call them from notebooks with Entra ID |
| **Databricks Vector Search vs Azure AI Search** | Either can serve RAG. AI Search with Foundry IQ is the default for Foundry agents; Databricks Vector Search when data and pipelines already live in Unity Catalog |
| **Fabric** | **Mirrored Azure Databricks catalog** (metadata mirroring, so Unity Catalog tables appear in Fabric through shortcuts with no copy); Power BI Direct Lake or DirectQuery on the mirrored data; OneLake access from Databricks via ABFS |
| **Purview** | Scan Unity Catalog metadata into Microsoft Purview |

## How it works on Azure

### Deploy a secure workspace (Bicep sketch)

```bicep
resource ws 'Microsoft.Databricks/workspaces@2024-05-01' = {
  name: 'dbw-contoso-prod'
  location: location
  sku: { name: 'premium' }
  properties: {
    managedResourceGroupId: subscriptionResourceId('Microsoft.Resources/resourceGroups', 'rg-dbw-contoso-prod-managed')
    publicNetworkAccess: 'Disabled'
    requiredNsgRules: 'NoAzureDatabricksRules'          // used with back-end Private Link
    parameters: {
      customVirtualNetworkId: { value: vnet.id }        // VNet injection
      customPublicSubnetName: { value: 'snet-dbw-host' }
      customPrivateSubnetName: { value: 'snet-dbw-container' }
      enableNoPublicIp: { value: true }                 // secure cluster connectivity
    }
  }
}
```

Add **front-end Private Link** (users to workspace) and **back-end Private Link** (compute plane to control plane), and for serverless use **network connectivity configurations** and private endpoints to your storage. Check API versions and property names against the template reference.

### Unity Catalog setup essentials

```sql
-- Storage access through the access connector's managed identity
CREATE STORAGE CREDENTIAL sc_prod WITH (AZURE_MANAGED_IDENTITY (ACCESS_CONNECTOR_ID = '/subscriptions/.../accessConnectors/ac-dbw-prod'));
CREATE EXTERNAL LOCATION el_landing URL 'abfss://landing@stcontosoprod.dfs.core.windows.net/' WITH (STORAGE CREDENTIAL sc_prod);

CREATE CATALOG prod_sales;
CREATE SCHEMA prod_sales.gold;
GRANT USE CATALOG ON CATALOG prod_sales TO `sales-analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_sales.gold TO `sales-analysts`;
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Databricks vs Fabric | Databricks for heavy, code-first engineering, advanced ML/MLOps, multicloud standardisation, fine-grained compute control. Fabric for SaaS simplicity, Power BI-centric analytics and capacity billing. Coexistence through mirroring and shortcuts is common |
| Serverless vs classic | Serverless for most notebooks, jobs, pipelines and SQL (fast start, less ops); classic for custom images, GPUs, specific libraries, or where serverless isn't available |
| Managed vs external tables | Managed by default (predictive optimisation, simpler governance); external when other engines must own the files or for existing paths |
| Workspace topology | Separate workspaces per environment (dev/test/prod), with one metastore per region and catalogs per domain and environment |
| Vector search location | AI Search for Foundry-centric agents and mixed content; Databricks Vector Search for Unity Catalog-centric ML teams |

## Hands-on

1. Create a Premium workspace (VNet injection optional) and confirm it's attached to a Unity Catalog metastore.
2. Create an access connector, storage credential and external location for an ADLS container.
3. Build a Lakeflow Spark Declarative Pipeline with bronze and silver tables and an expectation; run it on serverless.
4. Grant a group `SELECT` on a gold table and verify lineage in Catalog Explorer.
5. Create a Genie space on the gold table and ask questions in natural language.
6. In Fabric, create a **mirrored Azure Databricks catalog** and query the same table through the SQL analytics endpoint.

## Security, cost and reliability notes

- **Security:** Unity Catalog for all data access; no workspace-local Hive metastore for new work. Use access connectors (managed identities), not storage keys or SAS.
- **Security:** VNet injection, secure cluster connectivity (no public IP) and Private Link for regulated environments; enable IP access lists and Conditional Access.
- **Cost:** DBUs plus VMs. Use job clusters or serverless rather than all-purpose clusters for production; set auto-termination; use cluster policies to cap sizes; track spend with `system.billing` tables.
- **Cost:** idle SQL warehouses cost money. Set auto-stop.
- **Reliability:** Delta time travel and Unity Catalog `UNDROP` for recovery; plan cross-region disaster recovery deliberately (deep clone, Delta Sharing or replication of storage plus IaC for workspaces).

## Common mistakes

- Using all-purpose clusters for scheduled production jobs.
- Mounting storage with account keys instead of Unity Catalog external locations.
- One catalog for everything, with permissions granted to individuals instead of groups.
- Copying Databricks data into Fabric when mirroring would do.
- Assuming Databricks Mosaic AI and Foundry are mutually exclusive. They integrate in both directions.

## Check yourself

1. What runs in the control plane, and what runs in your subscription?
2. Describe Unity Catalog's three-level namespace.
3. What was Delta Live Tables renamed to, and what does it provide?
4. How does Databricks access ADLS without keys?
5. How can Fabric use Unity Catalog tables without copying data?
6. Name two ways Databricks and Foundry can be connected.

<details>
<summary>Answers</summary>

1. The control plane (web app, job scheduler, Unity Catalog services) runs in Databricks-managed infrastructure; classic compute clusters run in your Azure subscription (optionally your VNet). Serverless compute runs in Databricks-managed compute.
2. `catalog.schema.object`, where objects are tables, views, volumes, functions and models.
3. Lakeflow Spark Declarative Pipelines: declarative streaming tables and materialized views with expectations (data quality), dependency management and incremental processing.
4. Through a storage credential backed by an Azure Databricks access connector (managed identity), with external locations granting governed access.
5. With a mirrored Azure Databricks catalog in Fabric (metadata mirroring using shortcuts).
6. Foundry agents calling a Databricks Genie space as a tool or connection; Databricks Model Serving using Foundry or Azure OpenAI deployments as external models (or notebooks calling them with Entra ID).

</details>

## References

- [What is Azure Databricks?](https://learn.microsoft.com/azure/databricks/introduction/)
- [Azure Databricks architecture overview](https://learn.microsoft.com/azure/databricks/getting-started/overview)
- [What is Unity Catalog?](https://learn.microsoft.com/azure/databricks/data-governance/unity-catalog/)
- [Lakeflow Spark Declarative Pipelines](https://learn.microsoft.com/azure/databricks/ldp/)
- [What happened to DLT?](https://learn.microsoft.com/azure/databricks/ldp/where-is-dlt)
- [Mosaic AI Model Serving](https://learn.microsoft.com/azure/databricks/machine-learning/model-serving/)
- [Mosaic AI Vector Search](https://learn.microsoft.com/azure/databricks/generative-ai/vector-search)
- [Mirrored Azure Databricks catalog in Fabric](https://learn.microsoft.com/fabric/mirroring/azure-databricks)
- [Serverless compute](https://learn.microsoft.com/azure/databricks/compute/serverless/)
