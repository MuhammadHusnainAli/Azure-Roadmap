# Azure Database for PostgreSQL

> Topic: 09 · Databases · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

PostgreSQL has become the most popular open-source database for new applications, and for AI applications in particular, thanks to extensions such as pgvector. Azure Database for PostgreSQL flexible server provides managed PostgreSQL with high availability, backups, Entra authentication and Azure-specific AI extensions. For many AI engineers it is the simplest way to keep relational data, JSON and vectors in one place.

## Key concepts

### Deployment options

| Option | Status | Notes |
|---|---|---|
| **Flexible server** | The service | Single-node (with optional HA standby) managed PostgreSQL |
| **Elastic clusters** (on flexible server) | Available | Horizontal sharding based on Citus for multi-tenant and large datasets |
| Single server | **Retired** (March 2025) | Migrate to flexible server |
| Azure Cosmos DB for PostgreSQL | Not recommended for new projects | Use elastic clusters on flexible server instead |
| **Azure HorizonDB** | **Preview** | New cloud-native, PostgreSQL-compatible service with scale-out compute and shared storage for very large, demanding workloads. Evaluate for future designs; do not plan production on preview without checking status |

### Compute tiers

| Tier | VM family | Fit |
|---|---|---|
| **Burstable** | B-series | Dev, test, low-traffic apps; CPU credits |
| **General Purpose** | D-series | Most production workloads |
| **Memory Optimized** | E-series | Large working sets, analytics-heavy queries, vector indexes in memory |

Storage: Premium SSD (with autogrow) or **Premium SSD v2** (independently configurable IOPS and throughput). Compute can be scaled with a brief restart; storage can grow online.

### Supported versions

Multiple major PostgreSQL versions are supported, with new majors added after community release (recent ones include 16, 17 and 18). **In-place major version upgrade** is available. Plan upgrades before a version leaves support; extended support may be charged.

### High availability

| Mode | Standby location | Notes |
|---|---|---|
| **Zone-redundant HA** | Different zone in the same region | Recommended for production |
| Same-zone HA | Same zone | Where zones are unavailable |
| No HA | – | Dev and test |

HA uses synchronous replication to a standby; automatic failover typically completes in tens of seconds to a couple of minutes. The standby is not readable; use **read replicas** for reads.

### Read replicas and virtual endpoints

- Up to several **read replicas**, in the same or other regions (asynchronous).
- **Virtual endpoints** give stable names for the writer and a reader, so applications do not change connection strings when a replica is promoted.
- **Promote** a replica for regional DR (planned or forced); this is the main cross-region DR method.

### Extensions

PostgreSQL's strength is its extension ecosystem. Allow-list extensions with the `azure.extensions` server parameter, then `CREATE EXTENSION`.

| Extension | Use |
|---|---|
| **vector** (pgvector) | Vector data type, distance operators, HNSW and IVFFlat indexes |
| **pg_diskann** | Microsoft's DiskANN index for PostgreSQL: high recall at scale with lower memory than HNSW |
| **azure_ai** | Call Foundry and Azure OpenAI (embeddings, chat), Azure AI Language and Azure ML endpoints from SQL |
| **azure_local_ai** | Generate embeddings locally in the database for supported models (check current status) |
| **age** (Apache AGE) | Graph queries with openCypher; useful for GraphRAG patterns |
| **postgis** | Geospatial |
| **pg_cron** | Scheduled jobs |
| **pg_stat_statements**, **pg_hint_plan** | Query performance |
| **azure_storage** | Import and export with Blob Storage |
| **pg_partman** | Partition management |

### Built-in intelligence and tooling

- **Query Store** and **Intelligent tuning** (autovacuum and parameter recommendations), **index tuning** recommendations.
- **PgBouncer** built in for connection pooling (port 6432).
- Visual Studio Code PostgreSQL extension with Copilot integration.
- **Fabric mirroring** for PostgreSQL flexible server (check current status).

## How it works on Azure

### Connecting with Entra ID

```python
import psycopg
from azure.identity import DefaultAzureCredential

token = DefaultAzureCredential().get_token("https://ossrdbms-aad.database.windows.net/.default").token
conn = psycopg.connect(
    host="<server>.postgres.database.azure.com", dbname="app",
    user="app-identity-name",      # the Entra principal name mapped in PostgreSQL
    password=token, sslmode="require",
)
```

Tokens expire after about an hour; connection pools must refresh them (many libraries support a password callback).

Map a managed identity to a database role:

```sql
-- run as the Entra admin, connected to the postgres database
SELECT * FROM pgaadauth_create_principal('app-identity-name', false, false);
-- then in the app database
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "app-identity-name";
```

### pgvector and azure_ai together

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS azure_ai;

-- Configure azure_ai to call a Foundry endpoint (managed identity support depends on version; keys shown are an alternative)
SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<resource>.openai.azure.com');

CREATE TABLE docs (
  id bigserial PRIMARY KEY,
  dept text,
  content text,
  embedding vector(1536)
);

-- Generate embeddings in SQL
UPDATE docs SET embedding = azure_openai.create_embeddings('text-embedding-3-small', content)::vector
WHERE embedding IS NULL;

CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops);
-- or, with pg_diskann:  CREATE INDEX ON docs USING diskann (embedding vector_cosine_ops);

-- Hybrid: relational filter + vector similarity
SELECT id, content, embedding <=> azure_openai.create_embeddings('text-embedding-3-small', 'parental leave rules')::vector AS distance
FROM docs
WHERE dept = 'hr'
ORDER BY distance
LIMIT 5;
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Tier | Burstable for dev; General Purpose by default; Memory Optimized for large indexes and working sets |
| HA | Zone-redundant HA for production |
| Scaling reads | Read replicas with virtual endpoints |
| Scaling writes | Elastic clusters (Citus) for sharded multi-tenant data; evaluate HorizonDB as it matures |
| Vectors | pgvector HNSW for moderate scale with high recall; pg_diskann for larger datasets and lower memory |
| Embeddings | Generate in the application (more control) or with `azure_ai` in SQL (simpler pipelines) |
| Connection handling | Use the built-in PgBouncer for many short connections (serverless functions, many pods) |
| Storage | Premium SSD v2 when you need to tune IOPS independent of size |

## Hands-on

```bash
RG=rg-pg-lab; LOC=swedencentral; PG=pg-lab-$RANDOM
ME_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ME_ID=$(az ad signed-in-user show --query id -o tsv)
az group create -n $RG -l $LOC -o none

# 1. Burstable server with Entra auth only and public access for your IP (lab)
az postgres flexible-server create -g $RG -n $PG -l $LOC --tier Burstable --sku-name Standard_B1ms \
  --version 17 --storage-size 32 --microsoft-entra-auth Enabled --password-auth Disabled \
  --public-access $(curl -s ifconfig.me) -o none
az postgres flexible-server microsoft-entra-admin create -g $RG -s $PG -u $ME_UPN -i $ME_ID -t User -o none

# 2. Allow-list extensions
az postgres flexible-server parameter set -g $RG -s $PG -n azure.extensions --value "VECTOR,AZURE_AI,PG_DISKANN" -o none

# 3. Connect with an Entra token
export PGPASSWORD=$(az account get-access-token --resource-type oss-rdbms --query accessToken -o tsv)
psql "host=$PG.postgres.database.azure.com dbname=postgres user=$ME_UPN sslmode=require" -c "
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE items (id int PRIMARY KEY, label text, e vector(3));
INSERT INTO items VALUES (1,'cat','[1,0,0]'),(2,'kitten','[0.95,0.05,0]'),(3,'car','[0,0,1]');
CREATE INDEX ON items USING hnsw (e vector_cosine_ops);
SELECT label, e <=> '[1,0,0]' AS d FROM items ORDER BY d LIMIT 2;"

# 4. Add zone-redundant HA (needs General Purpose or higher; shown for reference)
# az postgres flexible-server update -g $RG -n $PG --tier GeneralPurpose --sku-name Standard_D2ds_v5 --high-availability ZoneRedundant

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** Entra authentication with password auth disabled; private access (VNet integration) or private endpoints; TLS required.
- **Security:** `azure_ai` stores endpoint settings in the database; prefer managed identity where supported, and restrict who can call the functions.
- **Cost:** HA doubles compute cost; read replicas each cost a full server. Stop dev servers when idle (`az postgres flexible-server stop`).
- **Cost:** reserved capacity for steady production compute.
- **Reliability:** zone-redundant HA; geo-redundant backups or cross-region read replicas for DR; test promotion.
- **Reliability:** vector indexes (especially HNSW) need memory; under-sized servers fall back to slow scans.

## Common mistakes

- Still running on retired Single Server.
- Password auth with shared admin credentials in app settings.
- Forgetting to allow-list extensions, then `CREATE EXTENSION` fails.
- No connection pooling for serverless or highly parallel clients, exhausting connections.
- Building HNSW indexes on a Burstable server and running out of memory.
- Using the HA standby for reads (it is not readable).

## Check yourself

1. What happened to Single Server, and what replaces Cosmos DB for PostgreSQL for new projects?
2. What does zone-redundant HA protect against, and is the standby readable?
3. What are virtual endpoints for?
4. Compare HNSW and DiskANN indexes for vectors in PostgreSQL.
5. What does the `azure_ai` extension let you do?
6. How does an application authenticate with Entra ID, and what must it handle?
7. What is Azure HorizonDB?

<details>
<summary>Answers</summary>

1. Single Server was retired in 2025; elastic clusters on flexible server replace Cosmos DB for PostgreSQL for new projects.
2. Failure of a zone or the primary node, with automatic failover to a synchronous standby in another zone. The standby is not readable.
3. Stable writer and reader endpoints that keep pointing to the right servers after replica promotion.
4. HNSW: high recall and speed, memory-hungry. DiskANN: designed for large datasets with high recall and lower memory, using disk-based graph indexes.
5. Call Foundry and Azure AI services (embeddings, chat completions, language features) and Azure ML endpoints from SQL.
6. It obtains an Entra token (for the `oss-rdbms` resource) and uses it as the password; it must refresh tokens before they expire, including in connection pools.
7. A new, PostgreSQL-compatible cloud-native database service in preview, with scale-out architecture for very large workloads.

</details>

## References

- [Azure Database for PostgreSQL flexible server overview](https://learn.microsoft.com/azure/postgresql/flexible-server/overview)
- [Compute and storage options](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-compute)
- [High availability](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-high-availability)
- [Read replicas and virtual endpoints](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-read-replicas)
- [pgvector on Azure PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector)
- [DiskANN for PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgdiskann)
- [azure_ai extension](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-azure-overview)
- [Elastic clusters](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-elastic-clusters)
- [Microsoft Entra authentication](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-azure-ad-authentication)
- [Azure HorizonDB](https://learn.microsoft.com/azure/horizondb/overview)
