# Azure DocumentDB

> Topic: 09 · Databases · Tracks: E ○ A ○ C ○ · Status checked: 2026-10

## Why it matters

MongoDB is one of the most widely used document databases, and many applications arrive at Azure built on it. **Azure DocumentDB** (formerly **Azure Cosmos DB for MongoDB vCore**) is a fully managed, MongoDB-compatible service built on the open-source DocumentDB engine. It gives MongoDB applications a familiar, vCore-priced home on Azure with vector search for AI, without the request-unit model of Cosmos DB. Knowing when to choose it, and how it differs from Cosmos DB for MongoDB (RU), avoids a common source of confusion.

## Key concepts

### What it is

| Aspect | Detail |
|---|---|
| Engine | Built on **DocumentDB**, an open-source document database engine on PostgreSQL, contributed by Microsoft and now governed in the Linux Foundation |
| Compatibility | MongoDB wire protocol and drivers; supports most MongoDB query language features, aggregation pipelines and indexes (check the compatibility page for specific operators) |
| Pricing | **vCore-based clusters** with storage: familiar to teams used to MongoDB Atlas or self-managed MongoDB |
| Scale | Vertical scaling of cluster tiers and horizontal **sharding** across shards |
| AI | **Vector search** (IVF, HNSW, DiskANN indexes) and text search within the MongoDB API |
| Free tier | A free cluster tier for learning and small projects |

### DocumentDB vs Cosmos DB for MongoDB (RU)

| | **Azure DocumentDB** | **Cosmos DB for MongoDB (RU)** |
|---|---|---|
| Pricing model | vCores and storage | Request units and storage |
| Compatibility | High; aimed at lift-and-shift of MongoDB apps | Good; some differences |
| Long or complex aggregations | Well suited | RU cost can climb |
| Serverless | No | Yes |
| Global distribution | Cross-region replicas for DR and reads | Turnkey global distribution, multi-region writes |
| SLA latency guarantees | Standard | Single-digit ms SLA |
| Vector search | Yes | Limited compared with DocumentDB |
| Fit | Migrations from MongoDB, apps needing rich queries and predictable vCore cost, RAG on MongoDB | Cloud-native, globally distributed, elastic apps that use MongoDB drivers |

**Rule of thumb:** migrating a MongoDB application → DocumentDB. Building a globally distributed, elastic app and want MongoDB drivers → Cosmos DB for MongoDB (RU), or Cosmos DB for NoSQL if you are not tied to MongoDB.

### Cluster configuration

| Setting | Detail |
|---|---|
| **Cluster tier** | vCores and RAM (for example M10, M20, M30 … up to large tiers; burstable tiers for dev) |
| Storage | Per shard, scales independently |
| **High availability** | Optional standby per shard, in another zone where available, with automatic failover |
| **Shards** | Horizontal scale-out; choose a shard key for large collections |
| **Replicas** | Cross-region read replicas for DR and read scaling; promote on failure |
| Networking | Public endpoint with firewall, or private endpoints |
| Authentication | Native MongoDB users (SCRAM) and **Microsoft Entra ID** authentication (check current status per driver) |
| Backups | Automatic with point-in-time restore |

### Vector search

```javascript
// Create a DiskANN vector index
db.runCommand({
  createIndexes: "products",
  indexes: [{
    name: "vectorIndex",
    key: { embedding: "cosmosSearch" },
    cosmosSearchOptions: { kind: "vector-diskann", dimensions: 1536, similarity: "COS" }
  }]
});

// Query: nearest neighbours with a filter
db.products.aggregate([
  { $search: { cosmosSearch: { path: "embedding", vector: queryVector, k: 5, filter: { category: "outdoor" } } } },
  { $project: { name: 1, score: { $meta: "searchScore" } } }
]);
```

Index kinds include `vector-ivf`, `vector-hnsw` and `vector-diskann`; filter support and syntax evolve, so check current documentation.

## How it works on Azure

### Migration from MongoDB

| Method | Use |
|---|---|
| **Azure DocumentDB migration extension for Visual Studio Code** (or Azure Data Studio successors) | Guided assessment and migration, online or offline |
| `mongodump` / `mongorestore` | Small databases, offline |
| Native change streams-based tools and partner tools | Online migration with catch-up |
| Assessment | Check unsupported operators and features against the compatibility list first |

Typical steps: assess compatibility → size the cluster (vCores and storage from current metrics) → migrate data → switch connection string (`mongodb+srv://<cluster>.mongocluster.cosmos.azure.com/`) → validate → decommission.

### Connecting

```python
from pymongo import MongoClient
# Native auth shown; for Entra auth use the OIDC mechanism supported by recent drivers
client = MongoClient("mongodb+srv://<user>:<password>@<cluster>.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false")
db = client["shop"]
db.products.insert_one({"name": "Trail tent", "category": "outdoor", "price": 199})
print(db.products.find_one({"category": "outdoor"}))
```

Store credentials in Key Vault, or use Entra authentication where your driver supports it.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| DocumentDB vs Cosmos DB (RU) | DocumentDB for MongoDB migrations and vCore predictability; Cosmos DB for global, elastic, serverless or multi-region-write needs |
| DocumentDB vs MongoDB Atlas on Azure | DocumentDB for a first-party Azure service and integrated billing; Atlas (Marketplace) for full MongoDB feature parity and existing Atlas tooling |
| HA | Enable for production |
| Sharding | Only when a collection outgrows a single shard's capacity or throughput; choose a shard key like a Cosmos DB partition key |
| Vector store | Use DocumentDB vectors when the app's data already lives there; Azure AI Search for large document RAG |

## Hands-on

```bash
RG=rg-docdb-lab; LOC=swedencentral; CL=docdb-lab-$RANDOM
az group create -n $RG -l $LOC -o none
az extension add -n cosmosdb-preview -y      # cluster commands may live in a preview extension

# Create a free-tier cluster (parameter names may change; check `az cosmosdb mongocluster create --help`)
az cosmosdb mongocluster create -g $RG -n $CL -l $LOC \
  --administrator-login labadmin --administrator-login-password "$(openssl rand -base64 18)Aa1!" \
  --server-version 7.0 --shard-node-tier Free --shard-node-disk-size-gb 32 --shard-node-count 1 -o none

az cosmosdb mongocluster firewall rule create -g $RG -c $CL -n me \
  --start-ip-address $(curl -s ifconfig.me) --end-ip-address $(curl -s ifconfig.me) -o none
```

Then connect with `mongosh`, create a collection with a vector index as above, insert a few documents with small embeddings, and run a `$search` query.

## Security, cost and reliability notes

- **Security:** private endpoints for production; Entra authentication where supported; never commit connection strings with passwords.
- **Cost:** vCore tiers are billed whether busy or not; right-size and use burstable or free tiers for dev.
- **Reliability:** enable HA per shard, plus a cross-region replica for DR; test promotion.

## Common mistakes

- Choosing Cosmos DB for MongoDB (RU) for a lift-and-shift app with heavy aggregations, then facing high RU costs.
- Assuming 100% MongoDB compatibility without checking operators the app uses.
- Sharding prematurely or with a poor shard key.
- Calling it "Cosmos DB" in designs and confusing it with the RU-based service. Use the current name.

## Check yourself

1. What was Azure DocumentDB previously called?
2. What engine is DocumentDB built on, and how is it governed?
3. Name three differences between DocumentDB and Cosmos DB for MongoDB (RU).
4. Which index kinds support vector search in DocumentDB?
5. What is the recommended first step in a MongoDB migration?

<details>
<summary>Answers</summary>

1. Azure Cosmos DB for MongoDB vCore.
2. The open-source DocumentDB engine built on PostgreSQL, governed under the Linux Foundation.
3. Any three of: vCore vs RU pricing; higher compatibility for lift-and-shift; no serverless; no multi-region writes; richer vector search; better fit for complex aggregations.
4. IVF, HNSW and DiskANN.
5. Assess compatibility of the application's features and operators against DocumentDB, then size the cluster.

</details>

## References

- [Azure DocumentDB overview](https://learn.microsoft.com/azure/documentdb/overview)
- [Compatibility with MongoDB](https://learn.microsoft.com/azure/documentdb/compatibility-features)
- [Vector search in DocumentDB](https://learn.microsoft.com/azure/documentdb/vector-search)
- [Choose between DocumentDB and Cosmos DB for MongoDB (RU)](https://learn.microsoft.com/azure/documentdb/choose-model)
- [Migrate MongoDB to Azure DocumentDB](https://learn.microsoft.com/azure/documentdb/migration-options)
- [DocumentDB open-source project](https://github.com/documentdb/documentdb)
