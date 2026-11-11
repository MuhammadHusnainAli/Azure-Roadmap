# RAG at Scale

## Why it matters

A RAG pilot over 500 documents for one team is easy. A RAG platform serving 200 customer tenants, 50 million chunks, sub-second latency and daily content changes is an architecture problem: tenant isolation, index partitioning, freshness, regional placement, embedding-model migrations and cost per query all become first-class concerns. Architects own these choices, and the wrong early decision (one shared index for every tenant with no filter discipline, or a tier that can't hold the vectors) is expensive to reverse.

## Key concepts

### Scale dimensions

| Dimension | Pressure point | Main levers |
|---|---|---|
| **Corpus size** | Storage and vector quota | Partitions, quantisation, dimension truncation, `stored: false`, L-tiers |
| **Query volume** | Throughput and latency | Replicas, caching, smaller models, fewer sub-queries |
| **Tenants** | Isolation and cost per tenant | Index-per-tenant, service-per-tenant, shared index with filters, hybrid |
| **Freshness** | Lag between source change and searchable | Indexer schedules, push updates, event-driven pipelines |
| **Geography** | Latency and residency | Regional services, routing, replication |
| **Change** | Schema and embedding-model upgrades | Index aliases, blue/green rebuilds |

### Multi-tenancy models (Azure AI Search)

| Model | Isolation | Density and cost | Notes |
|---|---|---|---|
| **One index per tenant** | Good (per-index boundary); use index-scoped RBAC if tenants connect directly | Shared service capacity; oversubscribe uneven tenants | Index count limits per tier; S3 HD supports many small indexes |
| **One service per tenant** | Highest (dedicated storage and throughput) | Highest cost per tenant; predictable | Services count against subscription limits; regional placement per tenant |
| **Hybrid** | Large tenants dedicated, long tail shared | Balanced | Needs a **tenant placement catalogue** for routing and moves |
| **Shared index with tenant filter** | Weakest; relies on the app applying `tenant_id` filters on every query | Highest density | Relevance statistics aren't tenant-scoped, so other tenants' documents affect scoring |

Microsoft's guidance for shared indexes: a trusted application layer must derive the tenant ID from **authenticated context** and apply the filter to every query; never accept a tenant ID from the client. Service partitions are **capacity units, not tenant-isolation boundaries**.

### Index lifecycle at scale

| Technique | Purpose |
|---|---|
| **Index aliases** | Apps query an alias; build `policies-v2` in parallel, then swap the alias for zero-downtime rebuilds |
| **Blue/green re-embedding** | Changing embedding model or chunking means a full re-index; build the new index alongside and compare on the golden set before swapping |
| **Incremental enrichment cache** | Avoid re-running expensive skills on unchanged documents |
| **Backup and restore samples** | Indexes have no built-in backup; rebuild from source or use Microsoft's backup/restore samples |

### Freshness patterns

| Pattern | Latency | Effort |
|---|---|---|
| Indexer schedule (minimum 5 minutes) | Minutes to hours | Low |
| On-demand indexer run triggered by an event (Event Grid → Function → `run_indexer`) | Minutes | Medium |
| Push API from the change feed (Cosmos DB change feed, Event Grid on Blob, Service Bus) | Seconds | Higher |
| Remote knowledge sources (query at request time) | Live | Low, but depends on the remote system |

Also plan **deletions** (soft delete detection) and **staleness signals** (freshness boosting, filters on `last_modified`, surfacing the document date in answers).

### Cost and latency anatomy of one RAG answer

| Component | Cost driver | Latency driver |
|---|---|---|
| Query rewriting / planning | LLM tokens | One LLM call |
| Embedding the query | Embedding tokens (small) | ~tens of ms |
| Search | Tier, replicas (fixed); semantic ranker or agentic retrieval tokens (variable) | Index size, filters, parallel sub-queries, reranking |
| Generation | Input tokens (chunks × size) + output tokens | Model size, output length, reasoning effort |

The largest variable cost is usually **generator input tokens**: halving chunks passed (with better ranking) often halves cost.

## How it works on Azure

| Need | Azure capability |
|---|---|
| Scale out queries | Replicas (up to 12 on Standard tiers; check limits) |
| Scale storage and vectors | Partitions; vector compression |
| Many tenants | S3 HD; index-per-tenant; index-scoped RBAC |
| Bursty or low-volume tenants | Serverless (Developer, Preview) tier, no SLA |
| Multi-region | Separate search services per region, with traffic routing (Front Door or app-level) and dual indexing; Azure AI Search is a single-region resource |
| AI gateway | API Management in front of models and search for quotas, caching, routing and per-tenant metering ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)) |
| Semantic caching | API Management semantic cache policies, or Azure Managed Redis vector cache ([Topic 09 file 08](../09-databases/08-azure-managed-redis.md)), scoped per tenant and permission set |
| Monitoring | Azure Monitor metrics (search latency, throttled queries, document count), diagnostic logs, Foundry tracing |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Tenancy model** | Start with index-per-tenant for B2B SaaS with moderate tenant counts; hybrid as tenants grow; shared index only for many tiny tenants with strict filter enforcement in a trusted layer. |
| **Partitions vs compression** | Try scalar quantisation and dimension truncation before adding partitions. |
| **Replicas** | Size for p95 latency at peak QPS, plus two for read SLA (three if indexing continuously). |
| **Freshness SLA** | Agree it with the business (for example "policy changes searchable within 15 minutes") and choose the pattern to match. |
| **Regions** | Place indexes near users and data-residency boundaries; for DR, run a secondary service and re-index from source or dual-write. |
| **Agentic retrieval at scale** | Token-based cost grows with sub-queries and sources; keep reasoning effort low and sources consolidated. |
| **Caching** | Cache answers only where content is not permission-sensitive, or include the permission scope in the cache key. |

## Hands-on

Design exercise plus a zero-downtime rebuild.

**1. Capacity estimate.** For 2 million chunks of 500 tokens with 1,536-dimension float32 vectors:

```text
raw vectors   = 2,000,000 × 1,536 × 4 bytes ≈ 12.3 GB (+ HNSW graph overhead)
with int8 SQ  ≈ 3.1 GB (+ overhead)      → fits one S1 partition's 35 GB vector quota comfortably
text + fields ≈ 2,000,000 × ~3 KB        ≈ 6 GB storage
```

Write down tier, partitions and replicas for 50 QPS at p95 < 1 s, then validate with a load test.

**2. Alias swap.**

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import SearchAlias

ic = SearchIndexClient("https://<search>.search.windows.net", DefaultAzureCredential())
ic.create_or_update_alias(SearchAlias(name="policies", indexes=["policies-v1"]))
# ... build and evaluate policies-v2 (new chunking or embedding model) ...
ic.create_or_update_alias(SearchAlias(name="policies", indexes=["policies-v2"]))   # instant cut-over
```

Applications query `policies` and never change. Keep v1 until v2 has run cleanly in production, then delete it.

**3. Tenant filter enforcement.** In the API layer, derive `tenant_id` from the validated token and append `tenant_id eq '<id>'` to every query, combined with any security filters; add an automated test that a query for tenant A never returns tenant B's documents.

## Security, cost and reliability notes

- **Tenant isolation is a security control**, not just a design pattern: test it, log it, and alert on anomalies.
- **Noisy neighbours**: in shared services, one tenant's indexing or heavy queries can throttle others; monitor per-index load and move heavy tenants out.
- **Quotas**: search services per subscription and region are limited; plan subscription design for service-per-tenant models.
- **Embedding-model retirement**: plan re-embedding before the embedding model retires ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)).
- **Cost visibility**: tag indexes and services per tenant or product; meter per-tenant tokens through the AI gateway.
- **DR**: indexes can be rebuilt from source; record how long a full rebuild takes, because that's your recovery time.

## Common mistakes

| Mistake | Fix |
|---|---|
| Shared index with client-supplied tenant ID | Derive from authenticated context |
| Treating partitions as tenant boundaries | They're capacity units only |
| In-place schema changes on live indexes | Build new index, swap alias |
| Adding partitions before trying compression | Quantise and truncate first |
| Global semantic cache across tenants | Scope by tenant and permissions |
| No measured rebuild time | Time a full re-index; it defines recovery time |

## Check yourself

1. Compare the four multi-tenancy models for isolation and cost.
2. Why aren't search partitions a tenant isolation boundary?
3. What does Microsoft say about the tenant identifier in a shared index?
4. How do index aliases enable zero-downtime rebuilds?
5. Name three freshness patterns and their typical latency.
6. Which part of a RAG answer usually dominates variable cost?
7. How would you provide multi-region resilience for Azure AI Search?
8. Why can a shared index distort relevance between tenants?

<details>
<summary>Answers</summary>

1. Index-per-tenant: good isolation, shared capacity; service-per-tenant: strongest isolation, highest cost; hybrid: dedicated for large tenants, shared for the rest; shared index: highest density, weakest isolation (filter-based).
2. They're capacity units that the service manages; you can't control document placement or use them for access control.
3. A trusted application layer must derive it from authenticated context and apply the filter on every query; never accept it from the client.
4. Apps query a stable alias; you build and validate a new index, then repoint the alias instantly.
5. Indexer schedules (minutes to hours), event-triggered indexer runs (minutes), push from change feeds (seconds); remote sources are live.
6. Generator input tokens (number and size of chunks passed to the model).
7. Deploy separate search services in two regions, index both (dual-write or re-index from source), and route traffic with Front Door or app logic.
8. Scoring statistics such as term frequencies aren't tenant-scoped, so other tenants' documents influence BM25 scores.

</details>

## References

- [Multitenant SaaS design patterns for Azure AI Search](https://learn.microsoft.com/azure/search/search-modeling-multitenant-saas-applications)
- [Estimate and manage capacity](https://learn.microsoft.com/azure/search/search-capacity-planning)
- [Service limits](https://learn.microsoft.com/azure/search/search-limits-quotas-capacity)
- [Index aliases](https://learn.microsoft.com/azure/search/search-how-to-alias)
- [Multi-region deployments in Azure AI Search](https://learn.microsoft.com/azure/search/search-multi-region)
- [Compress vectors with quantisation](https://learn.microsoft.com/azure/search/vector-search-how-to-quantization)
