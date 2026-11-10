# Database Decision Guide

## Why it matters

Choosing a database is one of the most expensive decisions to reverse. A wrong choice shows up later as performance ceilings, runaway cost, painful migrations or missing features. Azure offers more than a dozen managed database options, several of which overlap. This guide gives a structured way to choose, so that engineers, architects and consultants reach the same answer for the same requirements.

## Key concepts

### Data models

| Model | Shape | Strengths | Azure services |
|---|---|---|---|
| **Relational** | Tables, rows, schema, SQL, joins, ACID transactions | Integrity, complex queries, reporting, mature tooling | Azure SQL Database, SQL Managed Instance, SQL Server on VMs, PostgreSQL, MySQL, SQL database in Fabric |
| **Document** | JSON documents, flexible schema | Developer productivity, nested data, horizontal scale | Cosmos DB for NoSQL, Azure DocumentDB (MongoDB-compatible), Cosmos DB for MongoDB (RU) |
| **Key-value** | Key → value | Extreme speed and scale for simple lookups | Azure Managed Redis, Cosmos DB, Table Storage |
| **Wide-column** | Rows with dynamic columns, partitioned | Massive write throughput, time series | Cosmos DB for Apache Cassandra, Azure Managed Instance for Apache Cassandra |
| **Graph** | Nodes and edges | Relationship traversal | Cosmos DB for Apache Gremlin, PostgreSQL with Apache AGE, SQL Server graph tables |
| **In-memory** | Data in RAM | Sub-millisecond latency, caching, sessions | Azure Managed Redis |
| **Vector** | Embeddings with similarity search | Semantic search, RAG, memory | Vector support in most of the above, plus Azure AI Search (topic 17) |
| **Analytical** | Columnar, large scans | Aggregations over billions of rows | Fabric, Databricks, Synapse, Azure Data Explorer (topic 12) |
| **Time series and logs** | Append-heavy, time-indexed | Telemetry, logs | Azure Data Explorer, Fabric Real-Time Intelligence |

### Workload type

| | OLTP (operational) | OLAP (analytical) |
|---|---|---|
| Queries | Many small reads and writes | Few large scans and aggregations |
| Latency | Milliseconds | Seconds to minutes |
| Data | Current state | History |
| Services | SQL, PostgreSQL, MySQL, Cosmos DB, DocumentDB, Redis | Fabric, Databricks, Synapse, Data Explorer |

Do not run heavy analytics on the operational database. Replicate into an analytical store: Cosmos DB and Azure SQL support **Fabric mirroring**, which copies data into OneLake in near real time without ETL pipelines.

### The decision questions

1. **What is the data model?** Relational with integrity constraints, documents, key-value, graph?
2. **Is it a migration?** Existing SQL Server, Oracle, PostgreSQL, MySQL or MongoDB code and skills push you toward a compatible target.
3. **What scale and distribution?** Single region or global, writes in multiple regions, thousands or millions of operations per second?
4. **What latency?** Sub-millisecond (cache), single-digit milliseconds, or tens of milliseconds acceptable?
5. **What consistency and transactions?** Multi-row ACID transactions across entities, or per-document or per-partition?
6. **What query patterns?** Ad hoc joins and reporting, or known access patterns by key?
7. **What operational model?** Fully managed PaaS, or control over the engine and OS?
8. **What cost profile?** Steady or spiky; serverless; free tiers; licensing (Azure Hybrid Benefit).
9. **What AI needs?** Vector search, hybrid search, change feed for re-embedding, memory for agents.
10. **What compliance needs?** Encryption with customer keys, ledger, residency, confidential computing.

## How it works on Azure

### Selection matrix

| Requirement | First choice | Also consider |
|---|---|---|
| New relational app, SQL Server skills | **Azure SQL Database** | SQL database in Fabric for analytics-integrated apps |
| Lift-and-shift SQL Server with instance features (SQL Agent, cross-database queries, CLR, linked servers) | **Azure SQL Managed Instance** | SQL Server on VMs |
| Need OS access or unsupported SQL Server features, third-party apps certified on VMs | **SQL Server on Azure VMs** | Managed Instance |
| New relational app, open source preference, rich extensions (pgvector, PostGIS) | **Azure Database for PostgreSQL** | Azure HorizonDB (preview) for very large scale |
| Existing MySQL app (WordPress, PHP, LAMP) | **Azure Database for MySQL** | – |
| Globally distributed, low-latency, elastic NoSQL; agent memory at scale | **Azure Cosmos DB for NoSQL** | – |
| MongoDB application or skills | **Azure DocumentDB** | Cosmos DB for MongoDB (RU) for serverless or global multi-region writes |
| Cassandra workloads | **Cosmos DB for Apache Cassandra** or **Managed Instance for Apache Cassandra** | – |
| Caching, sessions, rate limiting, semantic cache, short-term agent memory | **Azure Managed Redis** | – |
| Simple, cheap key-value | **Table Storage** | Cosmos DB for Table |
| Graph traversal | Cosmos DB for Gremlin, or PostgreSQL with Apache AGE | A graph in Fabric (check status) |
| Vector search for RAG over documents | **Azure AI Search** / Foundry IQ | Vectors in your operational database (file 09) |
| Oracle workloads | **Oracle Database@Azure** (Exadata and Autonomous in Azure datacentres) | Migrate to PostgreSQL |
| Logs, telemetry, time series analytics | Azure Data Explorer / Fabric Eventhouse | – |

### Quick comparison of operational databases

| | Azure SQL DB | PostgreSQL Flexible | Cosmos DB NoSQL | DocumentDB | Managed Redis |
|---|---|---|---|---|---|
| Model | Relational | Relational (+JSONB) | Document | Document (MongoDB API) | Key-value, in-memory |
| Scale-out writes | Hyperscale (read scale-out); sharding by app | Elastic clusters (Citus) | Automatic partitioning | Sharding | Clustering |
| Multi-region writes | No (geo-replicas are read-only) | No | **Yes** | No (replicas) | Active geo-replication |
| Serverless | Yes | No (Burstable tier) | Yes | No | No |
| Vector search | Yes (vector type, DiskANN index) | Yes (pgvector, pg_diskann) | Yes (DiskANN) | Yes | Yes (RediSearch) |
| Typical latency | ms | ms | single-digit ms (SLA) | ms | sub-ms |
| Pricing basis | vCores or DTUs, storage | vCores, storage | Request units, storage | vCores, storage | Memory size and tier |

### Polyglot persistence

Real solutions often combine stores, each doing what it is good at:

```text
AI customer service app
 ├─ Azure SQL Database      orders, customers, payments (relational integrity)
 ├─ Cosmos DB for NoSQL     conversation threads, agent state, user preferences (scale, low latency)
 ├─ Azure Managed Redis     session cache, semantic cache, rate limits (speed)
 ├─ Azure AI Search         knowledge base for RAG (hybrid search, semantic ranking)
 └─ Fabric (mirroring)      analytics on all of the above
```

Keep the number of databases justified: each adds operations, security and cost.

## Design decisions and trade-offs

| Trade-off | Guidance |
|---|---|
| Familiarity vs fit | Team skills matter. A well-run PostgreSQL beats a misused Cosmos DB |
| Relational vs document | Relational when data is highly related and integrity matters; document when access is by entity and schema evolves fast |
| Single database vs polyglot | Start with one operational store plus a cache; add stores for clear needs |
| PaaS vs IaaS | PaaS unless a specific feature or certification forces VMs |
| Vectors in the database vs separate search | In the database when vectors are closely tied to operational data and scale is moderate; Azure AI Search for document retrieval, hybrid ranking and large corpora |
| Licensing | Azure Hybrid Benefit for SQL Server licences with Software Assurance; open source engines avoid licence cost |

## Hands-on

**Decision exercise.** For each scenario, choose a primary database (and any secondary store), and write two sentences of justification.

1. A retail company is moving a 3 TB SQL Server 2016 database that uses SQL Agent jobs, cross-database queries and CLR assemblies.
2. A start-up builds a multi-tenant SaaS app with a small team that prefers open source and needs geospatial queries.
3. A global gaming company needs player profiles readable and writable in under 10 ms in five regions.
4. An AI assistant needs to store chat threads for two million users, retrieve the last 20 messages quickly, and keep summaries as long-term memory.
5. A team has a Node.js app built on MongoDB Atlas and wants to move to Azure with minimal code changes.
6. An API needs to cache LLM responses for semantically similar questions.
7. A company has 40,000 PDF policies and wants a RAG assistant with hybrid search and permission trimming.

<details>
<summary>Suggested answers</summary>

1. Azure SQL Managed Instance: instance-level features (SQL Agent, cross-database queries, CLR) are supported with near-full compatibility.
2. Azure Database for PostgreSQL flexible server with PostGIS: open source, rich extensions, modest operational load.
3. Azure Cosmos DB for NoSQL with multi-region writes and a partition key on player ID: SLA-backed single-digit millisecond latency globally.
4. Cosmos DB for NoSQL partitioned by user or thread ID for threads and memory documents, optionally with vector search for memory recall, and Redis for hot session state.
5. Azure DocumentDB: MongoDB wire-protocol compatible, vCore-based, minimal code change.
6. Azure Managed Redis with vector search as a semantic cache (often via API Management's semantic caching policy).
7. Azure AI Search (or Foundry IQ) over Blob or Data Lake, with hybrid search, semantic ranker and document-level permissions.

</details>

## Security, cost and reliability notes

- **Security:** every option here supports Entra authentication and private endpoints; make both a selection criterion, not an afterthought.
- **Cost:** compare at realistic load. Cosmos DB RU costs depend heavily on data modelling; SQL costs on vCores and tier; Redis on memory size.
- **Cost:** serverless options (SQL Database serverless, Cosmos DB serverless) suit intermittent workloads; provisioned suits steady load.
- **Reliability:** check zone redundancy and multi-region options for each candidate against your RPO and RTO (file 12).

## Common mistakes

- Choosing Cosmos DB for a relational workload "because it scales", then struggling with joins and RU costs.
- Choosing SQL Database when the application needs instance features only Managed Instance has.
- Running analytics queries on the OLTP database.
- Adding a vector database product when the existing database or Azure AI Search already supports vectors.
- Ignoring team skills.
- Polyglot persistence without a reason for each store.

## Check yourself

1. Name the Azure service you would choose for: relational lift-and-shift with SQL Agent; global multi-region writes; MongoDB compatibility; sub-millisecond caching.
2. What is the difference between OLTP and OLAP, and why keep them separate?
3. What does Fabric mirroring offer?
4. When is SQL Server on VMs justified over Managed Instance?
5. When should vectors live in Azure AI Search rather than the operational database?
6. List five questions you ask before choosing a database.

<details>
<summary>Answers</summary>

1. Azure SQL Managed Instance; Azure Cosmos DB for NoSQL; Azure DocumentDB; Azure Managed Redis.
2. OLTP serves many small transactional operations on current data; OLAP serves large scans and aggregations on history. Mixing them degrades transactional performance and costs more.
3. Near real-time replication of operational database data into OneLake for analytics without building ETL pipelines.
4. When you need OS-level access, features or versions Managed Instance does not support, or a third-party application certified only on VMs.
5. For document retrieval over large corpora needing hybrid keyword and vector ranking, semantic reranking, indexers, chunking and permission trimming.
6. Any five of: data model, migration source, scale and distribution, latency, consistency and transactions, query patterns, operational model, cost profile, AI needs, compliance.

</details>

## References

- [Understand data store models](https://learn.microsoft.com/azure/architecture/guide/technology-choices/data-store-overview)
- [Choose a data store](https://learn.microsoft.com/azure/architecture/guide/technology-choices/data-store-decision-tree)
- [Azure databases overview](https://azure.microsoft.com/products/category/databases/)
- [Mirroring in Microsoft Fabric](https://learn.microsoft.com/fabric/database/mirrored-database/overview)
- [Oracle Database@Azure](https://learn.microsoft.com/azure/oracle/oracle-db/database-overview)
- [Choose an Azure service for vector search](https://learn.microsoft.com/azure/architecture/guide/technology-choices/vector-search)
