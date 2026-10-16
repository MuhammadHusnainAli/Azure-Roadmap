# Azure Managed Redis

> Topic: 09 · Databases · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Redis is the standard in-memory data store for caching, sessions, rate limiting, leaderboards, queues and pub/sub. In AI systems it also serves as a **semantic cache** for LLM responses and as fast short-term memory for agents. Azure Managed Redis is Microsoft's current Redis service, built on Redis Enterprise software, and is the migration target for the older Azure Cache for Redis, which has announced retirement dates. Engineers use it daily; architects and consultants need to plan the migration.

## Key concepts

### Azure Managed Redis vs Azure Cache for Redis

| | **Azure Managed Redis** | **Azure Cache for Redis** (Basic, Standard, Premium, Enterprise, Enterprise Flash) |
|---|---|---|
| Status | **Current service** | **Retiring**: Enterprise and Enterprise Flash tiers retire first (March 2027 announced), Basic, Standard and Premium later (September 2028 announced). New creation is being restricted; check the retirement page |
| Engine | Redis Enterprise software, recent Redis versions | Community Redis (Basic, Standard, Premium) or Redis Enterprise (Enterprise tiers) |
| Architecture | Multi-primary: every node runs primary shards, using all vCPUs | Primary/replica: replica node mostly idle |
| Modules | RediSearch (including vector search), RedisJSON, RedisBloom, RedisTimeSeries built in | Only Enterprise tiers |
| Active geo-replication | Yes (multi-region active-active) | Enterprise tiers only |
| Price-performance | Better at equivalent sizes in most cases | – |

**Plan migrations now.** Microsoft provides guidance for mapping Azure Cache for Redis tiers to Managed Redis SKUs.

### Tiers

| Tier | Memory-to-vCPU ratio | Fit |
|---|---|---|
| **Memory Optimized** | High memory per vCPU | Large datasets with moderate throughput (dev and test too) |
| **Balanced** | Balanced | Most production workloads; good default |
| **Compute Optimized** | More vCPU per GB | High throughput, many operations per second |
| **Flash Optimized** | RAM plus NVMe flash | Very large datasets where only hot keys need RAM speed |

Each tier comes in sizes measured in GB of memory. High availability (a replica of each shard on another node, across zones where available) is on by default and should stay on for production.

### Clustering policies

| Policy | Behaviour | Client requirement |
|---|---|---|
| **OSS cluster** | Clients connect to each shard directly; best performance | Cluster-aware client |
| **Enterprise cluster** | Single endpoint; proxy routes to shards | Any client; some multi-key commands restricted across slots |
| **Non-clustered** | Single shard (smaller sizes) | Any client; simplest compatibility for migrations |

### Persistence

| Option | Detail |
|---|---|
| None | Pure cache; data lost if all replicas fail |
| **RDB** snapshots | Periodic snapshots to disk |
| **AOF** | Append every write to a log (every second); smaller loss window |

Use persistence when Redis holds data you cannot quickly rebuild (sessions, agent short-term memory). For pure caches, rely on the source of truth.

### Common uses

| Pattern | How |
|---|---|
| **Cache-aside** | Read from cache; on miss, read from DB and populate with a TTL |
| Session store | Key per session with TTL |
| Rate limiting | Counters with `INCR` and `EXPIRE`, or sliding windows with sorted sets |
| Leaderboards | Sorted sets |
| Distributed locks | `SET key value NX PX ttl` (with care) |
| Pub/sub and streams | Lightweight messaging, event streams |
| **Semantic cache** | Store prompt embeddings and responses; on a new prompt, vector-search for a similar past prompt above a similarity threshold and return the cached answer |
| **Agent short-term memory** | Recent turns and working state per conversation with TTL; fast reads every turn |
| Vector store | Small to medium vector sets with RediSearch, when sub-millisecond lookup matters |

### Semantic caching

```text
Prompt → embed → vector search in Redis (similarity ≥ 0.95?)
   ├─ hit  → return cached completion (no LLM call, ~ms, ~no cost)
   └─ miss → call model → store {embedding, prompt, completion, ttl}
```

**Azure API Management** has built-in **semantic caching policies** (`azure-openai-semantic-cache-lookup` / `llm-semantic-cache-lookup` and `-store`) that use Azure Managed Redis as the cache, so applications get semantic caching without code. Choose thresholds carefully: too low returns wrong answers; include user or tenant context in cache partitioning to avoid leaking answers across users.

## How it works on Azure

### Connecting with Entra ID from Python

```python
import redis
from azure.identity import DefaultAzureCredential
from redis_entraid.cred_provider import create_from_default_azure_credential

credential_provider = create_from_default_azure_credential(("https://redis.azure.com/.default",))
r = redis.Redis(host="<name>.<region>.redis.azure.net", port=10000, ssl=True,
                credential_provider=credential_provider, decode_responses=True)

r.set("greeting", "hello", ex=300)
print(r.get("greeting"))
```

The `redis-entraid` package handles token refresh. Grant the identity a data access policy assignment on the cache. Disable access keys once all clients use Entra.

### Cache-aside in a few lines

```python
import json
def get_product(pid: str):
    key = f"product:{pid}"
    if (cached := r.get(key)) is not None:
        return json.loads(cached)
    product = db_load_product(pid)                 # source of truth
    r.set(key, json.dumps(product), ex=600)        # TTL bounds staleness
    return product
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Tier | Balanced by default; Memory Optimized for large, low-throughput data; Compute Optimized for high ops/sec; Flash for very large datasets |
| Clustering | OSS cluster with cluster-aware clients for performance; non-clustered or Enterprise for easy migration |
| Persistence | Off for pure caches; AOF for data you need to keep |
| Geo | Active geo-replication for multi-region apps needing local reads and writes |
| Semantic cache | Through API Management policies where possible; tune threshold and scope per tenant or user |
| Migration from Azure Cache for Redis | Map SKUs, test client compatibility (port 10000 and clustering policy), then migrate by dual-write, export/import (RDB) or rebuild cache |

## Hands-on

```bash
RG=rg-redis-lab; LOC=swedencentral; R=amr-lab-$RANDOM
az group create -n $RG -l $LOC -o none
az extension add -n redisenterprise -y

# 1. A small Balanced cache with Entra auth and keys disabled (verify current parameter names with --help)
az redisenterprise create -g $RG -n $R -l $LOC --sku Balanced_B0 \
  --clustering-policy EnterpriseCluster --access-keys-auth Disabled \
  --modules name=RediSearch -o none

# 2. Give yourself data access
ME=$(az ad signed-in-user show --query id -o tsv)
az redisenterprise database access-policy-assignment create -g $RG --cluster-name $R --database-name default \
  --access-policy-assignment-name me --access-policy-name default --object-id $ME -o none

az redisenterprise show -g $RG -n $R --query "{host:hostName, sku:sku.name}" -o table
az group delete -n $RG --yes --no-wait
```

Then run the Python sample. As an exercise, implement a semantic cache: embed prompts with a Foundry embedding model, store them with `FT.CREATE` and a vector field, and look up with `FT.SEARCH ... KNN`.

## Security, cost and reliability notes

- **Security:** Entra authentication with access keys disabled; private endpoints; TLS only.
- **Security:** semantic caches can leak one user's answer to another. Partition by tenant and user context, never cache personal data in shared keys, and set TTLs.
- **Cost:** memory size drives cost. Set TTLs and eviction policies; do not use Redis as an unbounded database.
- **Reliability:** keep HA on; enable zone redundancy; use persistence where data matters; use active geo-replication for multi-region.
- **Reliability:** design for cache failure: the application must still work (more slowly) if Redis is unavailable.

## Common mistakes

- Building new solutions on Azure Cache for Redis after its retirement announcement.
- Disabling HA in production to save money.
- No TTLs, so memory fills and evictions surprise you.
- A semantic cache threshold so low that unrelated questions get the same answer.
- Treating Redis as the only copy of important data without persistence.
- Using a non-cluster-aware client with the OSS cluster policy.

## Check yourself

1. Why should new solutions use Azure Managed Redis instead of Azure Cache for Redis?
2. Name the four Managed Redis tiers and when to use each.
3. What is the difference between OSS and Enterprise clustering policies?
4. Explain how a semantic cache works and two risks it introduces.
5. When should you enable persistence?
6. How does an application authenticate to Managed Redis without keys?

<details>
<summary>Answers</summary>

1. Azure Cache for Redis has announced retirement dates for all tiers, and Managed Redis offers better price-performance, modules, and active geo-replication.
2. Memory Optimized (large data, moderate throughput), Balanced (general production), Compute Optimized (high throughput), Flash Optimized (very large datasets with hot subset in RAM).
3. OSS lets cluster-aware clients connect directly to shards for best performance; Enterprise exposes one endpoint behind a proxy so any client works.
4. Prompts are embedded and compared with cached prompt embeddings; above a similarity threshold, the cached response is returned. Risks: wrong answers for dissimilar questions if the threshold is too low, and leaking answers across users or tenants if not partitioned.
5. When Redis holds data that cannot be easily rebuilt from a source of truth, such as sessions or agent working memory.
6. With Entra tokens via a credential provider (for example `redis-entraid` with `DefaultAzureCredential`) and an access policy assignment for the identity on the database.

</details>

## References

- [Azure Managed Redis overview](https://learn.microsoft.com/azure/redis/overview)
- [Choosing the right tier](https://learn.microsoft.com/azure/redis/overview#choosing-the-right-tier)
- [Clustering policies](https://learn.microsoft.com/azure/redis/architecture)
- [Migrate from Azure Cache for Redis](https://learn.microsoft.com/azure/redis/migrate/migrate-overview)
- [Azure Cache for Redis retirement FAQ](https://learn.microsoft.com/azure/azure-cache-for-redis/retirement-faq)
- [Microsoft Entra authentication for Managed Redis](https://learn.microsoft.com/azure/redis/entra-for-authentication)
- [Semantic caching with API Management](https://learn.microsoft.com/azure/api-management/azure-openai-enable-semantic-caching)
- [Vector search with Redis](https://learn.microsoft.com/azure/redis/tutorial-semantic-cache)
