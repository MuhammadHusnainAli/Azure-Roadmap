# Azure SQL Database

> Topic: 09 · Databases · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Azure SQL Database is Microsoft's fully managed relational database built on the latest SQL Server engine. It is the default relational choice on Azure for new applications and many modernised ones: patching, backups, high availability and much tuning are handled for you. It now also stores and searches vectors natively, calls models from T-SQL, and mirrors into Fabric, so it plays a growing role in AI applications that need relational data and retrieval together.

## Key concepts

### Deployment options

```text
Logical server  (myserver.database.windows.net: logins, firewall, Entra admin, auditing defaults)
 ├─ Single database  "orders"      ← own compute and storage
 ├─ Single database  "catalogue"
 └─ Elastic pool     "tenants"     ← shared compute for many databases
      ├─ tenant001
      └─ tenant002 …
```

A logical server is a management container, not a server you manage. Databases on one server can be in different tiers.

### Purchasing models

| Model | What you buy | Tiers |
|---|---|---|
| **vCore** (recommended) | vCores, memory per vCore by hardware, storage separately; Azure Hybrid Benefit and reservations apply | General Purpose, Business Critical, Hyperscale |
| **DTU** | Bundled compute, storage and I/O units | Basic, Standard, Premium |

DTU is simpler for small databases; vCore gives transparency, licence reuse and access to all features.

### Service tiers (vCore)

| | **General Purpose** | **Business Critical** | **Hyperscale** |
|---|---|---|---|
| Architecture | Compute separate from remote premium storage | Local SSD with Always On availability group of replicas | Distributed: compute, page servers, log service, storage |
| Max size | Up to about 4 TB | Up to about 4 TB | Up to 128 TB |
| Latency | Low single-digit ms I/O | Lowest (local SSD) | Low, with caching |
| Built-in read replica | No | Yes (one readable secondary) | Up to several HA replicas plus named replicas |
| Failover | Slower (compute reattaches storage) | Fast | Fast |
| Backups | Standard | Standard | Snapshot-based, near-instant restore regardless of size |
| Fit | Most business apps | Low latency, high transaction rate, in-memory OLTP | Large or fast-growing databases, rapid scale, read scale-out |

Hyperscale pricing has been reduced in recent years and it is increasingly the default for new databases that may grow; check pricing against General Purpose for your size.

### Compute tiers

| Tier | Behaviour | Fit |
|---|---|---|
| **Provisioned** | Fixed vCores, billed per hour | Steady workloads |
| **Serverless** | Auto-scales between min and max vCores, billed per second; can **auto-pause** when idle (General Purpose) | Intermittent, unpredictable workloads, dev and test |

Serverless resume after auto-pause takes time (often under a minute), so the first query waits. Hyperscale serverless scales but does not auto-pause in the same way (check current behaviour).

### Elastic pools

Many databases with variable, non-overlapping peaks share a pool of vCores. Typical for **multi-tenant SaaS** with a database per tenant. Size the pool for aggregate peak, set per-database min and max. Hyperscale elastic pools exist for larger tenants.

### Free offer

Each subscription can create a limited number of serverless General Purpose databases with a monthly free allowance of vCore seconds and storage. Ideal for learning and small projects.

### AI-relevant features

| Feature | Detail |
|---|---|
| **Vector data type** | `VECTOR(n)` column type storing embeddings efficiently |
| **Vector functions** | `VECTOR_DISTANCE` (cosine, Euclidean, dot) for exact search |
| **Vector index (DiskANN)** | Approximate nearest neighbour index (`CREATE VECTOR INDEX`, `VECTOR_SEARCH`); check current GA status in Azure SQL Database |
| **Call models from T-SQL** | `sp_invoke_external_rest_endpoint` and external model definitions (`CREATE EXTERNAL MODEL`, `AI_GENERATE_EMBEDDINGS`) to call Foundry endpoints with managed identity (check status) |
| **JSON** | Native `json` data type and functions |
| **Regular expressions, fuzzy matching** | Newer T-SQL functions useful for text processing |
| **Fabric mirroring** | Near real-time replication to OneLake |
| **Copilot in SQL** | Natural language to SQL and troubleshooting in the portal and tools |

### Business continuity

| Feature | Detail |
|---|---|
| Built-in HA | Every tier; zone redundancy available for all vCore tiers in supported regions |
| Automated backups | Full, differential and log backups; point-in-time restore 1–35 days; long-term retention up to 10 years |
| **Active geo-replication** | Up to four readable secondaries in any regions; manual failover |
| **Failover groups** | Group databases; listener endpoints that follow the primary; planned and forced failover |

Covered in depth in file 12.

## How it works on Azure

### Connecting with Entra ID (no passwords)

```python
import struct, pyodbc
from azure.identity import DefaultAzureCredential

token = DefaultAzureCredential().get_token("https://database.windows.net/.default").token
token_bytes = token.encode("utf-16-le")
token_struct = struct.pack(f"<I{len(token_bytes)}s", len(token_bytes), token_bytes)
SQL_COPT_SS_ACCESS_TOKEN = 1256

conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};Server=tcp:<server>.database.windows.net,1433;"
    "Database=<db>;Encrypt=yes;TrustServerCertificate=no;",
    attrs_before={SQL_COPT_SS_ACCESS_TOKEN: token_struct},
)
print(conn.execute("SELECT SUSER_SNAME()").fetchone())
```

Many drivers accept `Authentication=ActiveDirectoryDefault` (or `ActiveDirectoryMsi`) in the connection string instead of handling tokens yourself.

Create a database user for a managed identity:

```sql
CREATE USER [app-orders-identity] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [app-orders-identity];
ALTER ROLE db_datawriter ADD MEMBER [app-orders-identity];
```

### Vector search in T-SQL

```sql
CREATE TABLE dbo.Products (
    Id INT PRIMARY KEY,
    Name NVARCHAR(200),
    Description NVARCHAR(MAX),
    Embedding VECTOR(1536)
);

DECLARE @q VECTOR(1536) = CAST(@queryEmbeddingJson AS VECTOR(1536));
SELECT TOP (5) Id, Name, VECTOR_DISTANCE('cosine', Embedding, @q) AS Distance
FROM dbo.Products
WHERE Category = 'outdoor'
ORDER BY Distance;
```

Exact search scans all rows (fine for tens of thousands); use the vector index for larger tables.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| vCore or DTU | vCore for production and anything that might grow; DTU only for small, simple databases |
| Tier | General Purpose by default; Business Critical for lowest latency and in-memory OLTP; Hyperscale for size, growth, fast restore or read scale-out |
| Serverless | For intermittent or unpredictable load and non-production; provisioned for steady load and latency-sensitive apps |
| Multi-tenant SaaS | Database per tenant in elastic pools for isolation; shared database with tenant ID and row-level security for many small tenants |
| Zone redundancy | Enable for production |
| Cost | Reservations for steady provisioned compute; Azure Hybrid Benefit if you own licences |

## Hands-on

```bash
RG=rg-sql-lab; LOC=swedencentral; SRV=sql-lab-$RANDOM; DB=appdb
ME_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ME_ID=$(az ad signed-in-user show --query id -o tsv)
az group create -n $RG -l $LOC -o none

# 1. Logical server with Entra-only authentication
az sql server create -g $RG -n $SRV -l $LOC --enable-ad-only-auth \
  --external-admin-principal-type User --external-admin-name $ME_UPN --external-admin-sid $ME_ID -o none

# 2. A free-offer serverless database (or a small serverless one if the free offer is used up)
az sql db create -g $RG -s $SRV -n $DB --edition GeneralPurpose --compute-model Serverless \
  --family Gen5 --capacity 2 --auto-pause-delay 60 --use-free-limit --free-limit-exhaustion-behavior AutoPause -o none

# 3. Allow your IP (lab only; use private endpoints in production)
az sql server firewall-rule create -g $RG -s $SRV -n me --start-ip-address $(curl -s ifconfig.me) --end-ip-address $(curl -s ifconfig.me) -o none

# 4. Connect with Entra (sqlcmd go version supports --authentication-method)
sqlcmd -S $SRV.database.windows.net -d $DB --authentication-method ActiveDirectoryDefault \
  -Q "SELECT @@VERSION; SELECT SUSER_SNAME();"

# 5. Try the vector type
sqlcmd -S $SRV.database.windows.net -d $DB --authentication-method ActiveDirectoryDefault -Q "
CREATE TABLE dbo.V (Id INT PRIMARY KEY, E VECTOR(3));
INSERT dbo.V VALUES (1,'[1,0,0]'),(2,'[0.9,0.1,0]'),(3,'[0,1,0]');
DECLARE @q VECTOR(3) = '[1,0,0]';
SELECT Id, VECTOR_DISTANCE('cosine', E, @q) AS d FROM dbo.V ORDER BY d;"

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** Entra-only authentication, managed identities for apps, private endpoints with public access disabled, auditing and Defender for SQL (file 11).
- **Cost:** serverless minimum vCores and auto-pause delay drive cost; a database that never idles costs more on serverless than provisioned.
- **Cost:** Business Critical costs roughly two to three times General Purpose for the same vCores; justify it.
- **Reliability:** enable zone redundancy; use failover groups for regional DR; test restores.
- **Reliability:** handle transient errors with retry logic; failovers and reconfigurations drop connections briefly.

## Common mistakes

- SQL authentication with passwords in configuration instead of Entra and managed identities.
- Opening the firewall to `0.0.0.0–255.255.255.255` or allowing all Azure services broadly.
- Choosing Business Critical for every database by default.
- Serverless for latency-sensitive production APIs that cannot tolerate resume delays.
- No retry logic for transient faults.
- Running reports on the primary instead of a read replica or Fabric mirror.

## Check yourself

1. What is a logical server and what does it hold?
2. Compare General Purpose, Business Critical and Hyperscale in one line each.
3. When does serverless save money, and when does it not?
4. What problem do elastic pools solve?
5. How does an app connect to Azure SQL without a password?
6. What is the difference between `VECTOR_DISTANCE` and a vector index?
7. Which feature provides automatic endpoint redirection during regional failover?

<details>
<summary>Answers</summary>

1. A management container with a DNS name that holds logins, the Entra admin, firewall rules and default policies for its databases.
2. General Purpose: remote storage, balanced price. Business Critical: local SSD with replicas, lowest latency and a readable secondary. Hyperscale: distributed architecture up to 128 TB with fast scaling, restores and read scale-out.
3. It saves when workloads are intermittent and can auto-pause or run at low vCores; it does not when load is steady and high.
4. Sharing compute between many databases with variable, non-overlapping peaks, typically multi-tenant SaaS.
5. With an Entra token, for example from a managed identity via `DefaultAzureCredential` or `Authentication=ActiveDirectoryDefault`, after creating a user `FROM EXTERNAL PROVIDER`.
6. `VECTOR_DISTANCE` computes exact distances (scanning rows); a vector index (DiskANN) gives approximate nearest neighbour search that scales to large tables.
7. Failover groups, through their read-write and read-only listener endpoints.

</details>

## References

- [What is Azure SQL Database?](https://learn.microsoft.com/azure/azure-sql/database/sql-database-paas-overview)
- [vCore purchasing model](https://learn.microsoft.com/azure/azure-sql/database/service-tiers-sql-database-vcore)
- [Hyperscale service tier](https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale)
- [Serverless compute tier](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview)
- [Elastic pools](https://learn.microsoft.com/azure/azure-sql/database/elastic-pool-overview)
- [Free offer](https://learn.microsoft.com/azure/azure-sql/database/free-offer)
- [Vector data type and functions](https://learn.microsoft.com/sql/relational-databases/vectors/vectors-sql-server)
- [Intelligent applications with Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/ai-artificial-intelligence-intelligent-applications)
- [Microsoft Entra authentication for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview)
