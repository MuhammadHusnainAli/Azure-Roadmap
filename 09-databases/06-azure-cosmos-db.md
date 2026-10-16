# Azure Cosmos DB

> Topic: 09 · Databases · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Azure Cosmos DB is Azure's globally distributed, multi-model NoSQL database, with SLA-backed single-digit millisecond latency, elastic scale and multi-region writes. It powers large consumer applications and is the default store for chat history and agent state in many AI systems, including Foundry Agent Service's standard setup. Its power comes with a modelling discipline: partition keys and request units decide performance and cost, and a wrong choice is expensive to fix.

## Key concepts

### Resource model and APIs

```text
Account  (regions, consistency default, network, keys or Entra)
 └─ Database  (optional shared throughput)
     └─ Container  (partition key, indexing policy, TTL, throughput, vector and full-text policies)
         └─ Items  (JSON documents, max 2 MB)
```

| API | Data model | Use |
|---|---|---|
| **NoSQL** | JSON documents, SQL-like queries | **Default for new apps**; gets new features first (vector, full-text, hybrid search) |
| MongoDB (RU) | MongoDB wire protocol | MongoDB apps needing serverless or multi-region writes (for vCore-based MongoDB see DocumentDB, file 07) |
| Apache Cassandra | Wide-column, CQL | Cassandra workloads |
| Apache Gremlin | Graph | Graph traversal |
| Table | Key-value | Upgrade path from Table Storage |

The API is chosen per account and cannot be changed.

### Partitioning

Data is split into **logical partitions** by the **partition key** value; logical partitions are spread across **physical partitions** that Cosmos DB manages.

| Rule | Detail |
|---|---|
| Logical partition limit | 20 GB of data per partition key value |
| Physical partition throughput | Up to 10,000 RU/s each |
| Good partition key | **High cardinality**, **even distribution** of storage and requests, and **used in most queries** |
| **Hierarchical partition keys** | Up to three levels (for example `tenantId` → `userId` → `sessionId`) to avoid the 20 GB limit for large tenants while keeping tenant-scoped queries efficient |
| Cross-partition queries | Fan out to every physical partition; more RU and latency |
| Changing the key | Requires copying data to a new container (container copy jobs help) |

Examples:

| Workload | Partition key |
|---|---|
| Chat threads | `/userId` or hierarchical `/tenantId`, `/userId` |
| Orders | `/customerId` |
| IoT telemetry | `/deviceId` (with time-based containers or TTL) |
| Multi-tenant SaaS | `/tenantId` (hierarchical if tenants vary widely in size) |
| Bad choices | `/country` (low cardinality, hot partitions), `/createdDate` (all writes to today) |

### Request units (RU)

Every operation costs **request units**, a normalised measure of CPU, memory and I/O. A 1 KB point read by ID and partition key costs 1 RU; a 1 KB write costs about 5 RU; queries cost more depending on complexity and index use. The `x-ms-request-charge` header tells you the cost of every operation.

| Capacity mode | How it works | Fit |
|---|---|---|
| **Provisioned (manual)** | Fixed RU/s per container or database, billed hourly | Steady, predictable load |
| **Autoscale** | Scales between 10% and 100% of a max RU/s, billed on the highest per hour | Variable load; most production apps |
| **Serverless** | Pay per RU consumed, no provisioning | Intermittent, dev, low traffic (single region) |
| **Free tier** | First 1,000 RU/s and 25 GB free per account (one per subscription) | Learning and small apps |

When demand exceeds provisioned RU/s, requests get **HTTP 429** with a retry-after; SDKs retry automatically. **Dynamic scaling** (per-region and per-partition autoscale) and **burst capacity** reduce waste and throttling for uneven loads. **Priority-based execution** lets low-priority requests be throttled first. **Throughput buckets** cap how much RU a workload can consume within a container (check status).

### Indexing

By default **every property is indexed** automatically. Tune the **indexing policy** to exclude large or unqueried paths (reducing write RU) and to add composite indexes for multi-property `ORDER BY` and filters. Vector and full-text indexes are declared in the policy.

### Consistency levels

| Level | Guarantee | Cost and latency |
|---|---|---|
| **Strong** | Linearisable reads; always latest committed write | Highest latency; reads cost 2× RU; not available with multi-region writes |
| **Bounded staleness** | Reads lag by at most K versions or T time | High |
| **Session** (default) | Read-your-own-writes within a session; monotonic reads | Low; best balance |
| **Consistent prefix** | Never see out-of-order writes | Lower |
| **Eventual** | No ordering guarantee | Lowest |

Session consistency suits almost all applications, including chat: a user always sees their own messages.

### Global distribution

- Add or remove regions at any time; data replicates automatically.
- **Single-region writes** with read regions, or **multi-region writes** (active-active) with conflict resolution (last writer wins by default, or custom).
- **Service-managed failover** and **per-partition automatic failover** for write region outages (check current status for per-partition automatic failover).
- Availability SLAs up to 99.999% with multi-region writes.

### Change feed

An ordered, persistent log of changes per partition. Uses:

- Trigger Azure Functions or other processors on inserts and updates.
- **Re-embed** documents when content changes and update a search index.
- Maintain materialised views and denormalised copies.
- Stream to analytics. **All versions and deletes** mode captures deletes and intermediate versions (requires continuous backup).

### AI features

| Feature | Detail |
|---|---|
| **Vector search** | Vectors stored in documents, with `flat`, `quantizedFlat` and **DiskANN** indexes; `VectorDistance()` in queries; filters combined in one query |
| **Full-text search** | BM25-ranked full-text indexes and `FullTextContains`, `FullTextScore` |
| **Hybrid search** | `ORDER BY RANK RRF(VectorDistance(...), FullTextScore(...))` to fuse vector and keyword relevance |
| **Fleets** | Manage many accounts (for example per tenant) as one fleet with shared throughput pools (check status) |
| **Cosmos DB in Fabric** | Cosmos DB databases created and managed inside Fabric, auto-mirrored to OneLake (check status) |
| **Agent Service** | Foundry standard setup stores threads and messages in your Cosmos DB account |

## How it works on Azure

### Python with Entra authentication

```python
from azure.identity import DefaultAzureCredential
from azure.cosmos import CosmosClient, PartitionKey

client = CosmosClient("https://<account>.documents.azure.com:443/", credential=DefaultAzureCredential())
db = client.get_database_client("chat")
threads = db.get_container_client("messages")

threads.upsert_item({
    "id": "msg-0001", "userId": "u-42", "threadId": "t-7",
    "role": "user", "content": "How do I reset my VPN token?", "ts": "2026-10-15T09:12:00Z",
})

# Point read: cheapest operation (≈1 RU)
item = threads.read_item("msg-0001", partition_key="u-42")

# Single-partition query
items = threads.query_items(
    "SELECT TOP 20 * FROM c WHERE c.threadId = @t ORDER BY c.ts DESC",
    parameters=[{"name": "@t", "value": "t-7"}], partition_key="u-42",
)
for m in items:
    print(m["role"], m["content"])

print(threads.client_connection.last_response_headers["x-ms-request-charge"])
```

Data-plane RBAC uses Cosmos DB's own **SQL role definitions** (`Cosmos DB Built-in Data Contributor`, `Reader`) assigned with `az cosmosdb sql role assignment create`, separate from Azure control-plane roles. Disable key-based auth with `disableLocalAuth: true`.

### Vector container definition (excerpt)

```json
{
  "vectorEmbeddingPolicy": { "vectorEmbeddings": [ { "path": "/embedding", "dataType": "float32", "distanceFunction": "cosine", "dimensions": 1536 } ] },
  "indexingPolicy": {
    "vectorIndexes": [ { "path": "/embedding", "type": "diskANN" } ],
    "excludedPaths": [ { "path": "/embedding/*" } ]
  }
}
```

```sql
SELECT TOP 5 c.id, c.content, VectorDistance(c.embedding, @q) AS score
FROM c
WHERE c.tenantId = @tenant
ORDER BY VectorDistance(c.embedding, @q)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| API | NoSQL for new work; others for compatibility |
| Partition key | Design from the top queries; high cardinality and even load; hierarchical for multi-tenant |
| Modelling | Embed related data read together; reference data that grows unbounded or changes independently; denormalise for reads and use change feed to keep copies in sync |
| Capacity mode | Autoscale for production; serverless for spiky low-volume or dev; provisioned for steady high load with reservations |
| Consistency | Session unless a specific need demands stronger or weaker |
| Regions | Add read regions near users; multi-region writes only when writes must be local everywhere or 99.999% is required |
| Indexing | Exclude large unqueried paths (including raw text and vectors from the range index) |
| TTL | Set TTL on transient data (sessions, telemetry, temporary agent state) |

## Hands-on

```bash
RG=rg-cosmos-lab; LOC=swedencentral; ACC=cosmos-lab-$RANDOM
az group create -n $RG -l $LOC -o none

# 1. Serverless NoSQL account with local auth disabled and vector search capability
az cosmosdb create -g $RG -n $ACC --locations regionName=$LOC failoverPriority=0 isZoneRedundant=false \
  --capabilities EnableServerless EnableNoSQLVectorSearch --disable-local-auth true -o none

# 2. Data-plane role for yourself
ME=$(az ad signed-in-user show --query id -o tsv)
az cosmosdb sql role assignment create -g $RG -a $ACC --role-definition-id 00000000-0000-0000-0000-000000000002 \
  --principal-id $ME --scope "/" -o none

# 3. Database and a container with a hierarchical partition key
az cosmosdb sql database create -g $RG -a $ACC -n chat -o none
az cosmosdb sql container create -g $RG -a $ACC -d chat -n messages \
  --partition-key-path "/tenantId" "/userId" --partition-key-kind MultiHash --partition-key-version 2 --ttl -1 -o none

az group delete -n $RG --yes --no-wait
```

Then run the Python sample against the account. Compare the RU charge of a point read, a single-partition query and a cross-partition query (`enable_cross_partition_query=True` without `partition_key`).

## Security, cost and reliability notes

- **Security:** disable local (key) auth; use Entra data-plane roles; private endpoints; customer-managed keys if required.
- **Cost:** RU consumption is driven by data modelling. Point reads beat queries; single-partition queries beat cross-partition ones; trimmed indexing reduces write cost.
- **Cost:** each added region multiplies RU and storage cost by the number of regions.
- **Cost:** free tier and serverless for small workloads; reservations for large provisioned throughput.
- **Reliability:** enable availability zones in each region; add a second region for DR; choose service-managed failover. Continuous backup enables point-in-time restore (file 12).
- **Reliability:** handle 429s with SDK retries and monitor normalised RU consumption per partition to spot hot partitions.

## Common mistakes

- Partition key with low cardinality or one that concentrates writes (dates, booleans, status).
- Treating Cosmos DB like a relational database: joins across containers, cross-partition queries everywhere.
- Using `id` as the partition key without thinking about query patterns (fine for pure key-value; poor for range queries).
- Indexing every path, including large text blobs and vectors in the range index.
- Strong consistency by default, doubling read cost and latency.
- Using account keys in application settings.
- Unbounded growth of items inside a single document (for example all messages in one thread document) hitting the 2 MB limit.

## Check yourself

1. What three properties make a good partition key?
2. What are hierarchical partition keys for?
3. What is a request unit, and roughly what do a 1 KB point read and write cost?
4. Compare provisioned, autoscale and serverless throughput.
5. What does session consistency guarantee and why is it the default?
6. Name three uses of the change feed in AI systems.
7. How does hybrid search work in Cosmos DB for NoSQL?
8. How is data-plane access controlled with Entra ID?

<details>
<summary>Answers</summary>

1. High cardinality, even distribution of storage and requests, and inclusion in most queries' filters.
2. Sub-partitioning by up to three levels so large tenants exceed the 20 GB logical partition limit while queries scoped to a tenant remain efficient.
3. A normalised cost unit for operations. About 1 RU for a 1 KB point read and about 5 RU for a 1 KB write.
4. Provisioned: fixed RU/s billed hourly for steady load. Autoscale: scales between 10% and 100% of a maximum, billed on the hourly peak, for variable load. Serverless: pay per RU consumed, for intermittent or low-volume use.
5. A client reads its own writes and sees monotonic reads within a session; it gives intuitive behaviour at low cost and latency.
6. Re-embedding changed documents and updating vector or search indexes; triggering agent or workflow actions on new data; building materialised views or streaming to analytics.
7. It combines vector similarity (`VectorDistance`) and BM25 full-text scores (`FullTextScore`) with reciprocal rank fusion (`RANK RRF`).
8. With Cosmos DB data-plane role definitions and assignments (such as Built-in Data Contributor), separate from Azure control-plane RBAC, and with local auth disabled.

</details>

## References

- [Azure Cosmos DB overview](https://learn.microsoft.com/azure/cosmos-db/introduction)
- [Partitioning and horizontal scaling](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview)
- [Hierarchical partition keys](https://learn.microsoft.com/azure/cosmos-db/hierarchical-partition-keys)
- [Request units](https://learn.microsoft.com/azure/cosmos-db/request-units)
- [Consistency levels](https://learn.microsoft.com/azure/cosmos-db/consistency-levels)
- [Global distribution](https://learn.microsoft.com/azure/cosmos-db/distribute-data-globally)
- [Change feed](https://learn.microsoft.com/azure/cosmos-db/change-feed)
- [Vector search in Cosmos DB for NoSQL](https://learn.microsoft.com/azure/cosmos-db/nosql/vector-search)
- [Hybrid search](https://learn.microsoft.com/azure/cosmos-db/gen-ai/hybrid-search)
- [Data-plane RBAC](https://learn.microsoft.com/azure/cosmos-db/nosql/security/how-to-grant-data-plane-role-based-access)
- [Data modelling in Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/nosql/modeling-data)
