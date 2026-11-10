# Vector Search in Azure Databases

## Why it matters

Embeddings turn text, images and other data into vectors so that "similar meaning" becomes "nearby in space". Vector search underpins RAG, semantic search, recommendations, deduplication, semantic caching and agent memory. Nearly every Azure database now stores and searches vectors, and Azure AI Search does too. The question is no longer "which vector database?" but "where should these vectors live?", and the answer affects consistency, security, cost and retrieval quality.

## Key concepts

### Vector search basics

| Concept | Detail |
|---|---|
| **Embedding** | A fixed-length array of floats from an embedding model (for example 1,536 or 3,072 dimensions) |
| **Distance metrics** | Cosine (most common for text), dot product (for normalised vectors), Euclidean (L2) |
| **Exact (k-NN, flat) search** | Compare the query to every vector; perfect recall, cost grows linearly |
| **Approximate (ANN) search** | Index structure finds near neighbours quickly; trades a little recall for big speed gains |
| **Recall** | Fraction of true nearest neighbours returned by ANN |
| **Filtering** | Restricting candidates by metadata (tenant, department, date); pre-filtering vs post-filtering affects correctness and speed |
| **Hybrid search** | Combine vector similarity with keyword (BM25) ranking, often fused with reciprocal rank fusion (RRF) |
| **Quantisation** | Store vectors in fewer bits (scalar or binary) to cut memory and cost; may reduce recall, often recovered by rescoring |
| **Dimensions** | More dimensions can be more accurate but cost more storage and compute; some models support shortening (Matryoshka embeddings) |

### ANN index types

| Index | How it works | Notes |
|---|---|---|
| **HNSW** | Multi-layer proximity graph | Fast, high recall; memory-heavy; slower builds |
| **IVF / IVFFlat** | Cluster vectors, search the nearest clusters | Needs training on data; recall depends on probes |
| **DiskANN** | Graph index designed to live largely on SSD, from Microsoft Research | High recall at large scale with low memory; supports filtered search well |
| **Quantized flat** | Exact scan over compressed vectors | Good for small to medium sets |

### Vector support across Azure

| Service | Vector capability | Index types | Hybrid / full-text | Notes |
|---|---|---|---|---|
| **Azure AI Search** | Vector fields, integrated vectorisation (chunk and embed during indexing) | HNSW, exhaustive k-NN; compression (scalar, binary) | **Yes**, plus **semantic ranker** | Purpose-built retrieval engine; Foundry IQ builds on it (topic 17) |
| **Azure SQL Database / SQL Server 2025** | `VECTOR` type, `VECTOR_DISTANCE`, vector index | DiskANN (check status) | Full-text search exists separately | Vectors next to relational data |
| **PostgreSQL flexible server** | pgvector | HNSW, IVFFlat, **DiskANN** (pg_diskann) | Full-text via `tsvector`; hybrid in SQL | Rich SQL filtering; `azure_ai` for embeddings |
| **Cosmos DB for NoSQL** | Vector policies and indexes | flat, quantizedFlat, **DiskANN** | **Yes** (BM25 full-text, RRF) | Global scale, multi-tenant, change feed |
| **Azure DocumentDB** | MongoDB `$search` with vectors | IVF, HNSW, DiskANN | Text indexes | For MongoDB apps |
| **Azure Managed Redis** | RediSearch vector fields | HNSW, flat | Text search | Sub-millisecond; semantic cache; smaller sets |
| **Fabric / Databricks** | Vector search features in the analytics platforms | Varies | Varies | For analytics-centred retrieval |

### Where should the vectors live?

| Put vectors in **Azure AI Search** when | Put vectors in the **operational database** when |
|---|---|
| Retrieving over documents (PDFs, pages, SharePoint, Blob corpora) | Vectors describe operational entities (products, tickets, user memories) that change with the data |
| You need hybrid ranking plus semantic reranking for answer quality | You need transactional consistency between data and its embedding |
| You want indexers, chunking, integrated vectorisation, skills | Queries combine vector similarity with relational or document filters and joins |
| Permission-aware retrieval over many sources | Scale is moderate and adding another service is unjustified |
| Many indexes and corpora managed centrally, Foundry IQ knowledge bases | Multi-tenant data already partitioned in Cosmos DB |
| Retrieval quality is the product | Latency and simplicity matter more than advanced ranking |

Many systems use both: Azure AI Search for knowledge documents, and the operational database for entity and memory vectors.

### Keeping embeddings fresh

| Trigger | Mechanism |
|---|---|
| Row or document changes | Cosmos DB change feed, SQL change tracking or change data capture, PostgreSQL logical replication or triggers → re-embed |
| New embedding model | Re-embed everything into a new field or index, switch over, then drop the old one. **Never mix vectors from different models** in one index |
| Deletes | Propagate to every index |

Store the embedding model name and version alongside vectors.

## How it works on Azure

### Generating embeddings (keyless)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<foundry-resource>.openai.azure.com/openai/v1/", api_key=tp)

def embed(texts: list[str]) -> list[list[float]]:
    resp = client.embeddings.create(model="<embedding-deployment>", input=texts)
    return [d.embedding for d in resp.data]
```

Batch inputs, respect rate limits, and cache embeddings for unchanged content.

### The same query in three databases

```sql
-- Azure SQL
SELECT TOP (5) Id, Title
FROM dbo.Tickets
WHERE TenantId = @tenant
ORDER BY VECTOR_DISTANCE('cosine', Embedding, @q);

-- PostgreSQL (pgvector)
SELECT id, title FROM tickets
WHERE tenant_id = $1
ORDER BY embedding <=> $2
LIMIT 5;

-- Cosmos DB for NoSQL (hybrid)
SELECT TOP 5 c.id, c.title FROM c
WHERE c.tenantId = @tenant
ORDER BY RANK RRF(VectorDistance(c.embedding, @q), FullTextScore(c.title, @keywords))
```

### Measuring retrieval quality

Build a small **golden set**: 50 to 200 queries with the documents that should be returned. Measure **recall@k** and **MRR** (mean reciprocal rank) for each configuration (index type, parameters, hybrid on or off, chunking). Change one variable at a time. Retrieval quality usually matters more to answer quality than the choice of language model (topic 17 and topic 22).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Exact or ANN | Exact (flat) up to tens of thousands of vectors per filter scope; ANN beyond |
| Index type | DiskANN where available for large or filtered workloads; HNSW for memory-resident speed; IVF when you can train and tune |
| Hybrid | Usually improves relevance for natural-language queries containing names, codes or rare terms. Default to hybrid for search over text |
| Dimensions and quantisation | Smaller or quantised vectors for large corpora; validate recall on your golden set |
| Multi-tenant | Filter by tenant in every query; partition data by tenant (Cosmos DB partition key, PostgreSQL row-level security) |
| Dedicated vector database products | Rarely needed on Azure; existing databases and Azure AI Search cover most needs |

## Hands-on

1. Using PostgreSQL from file 04 (or Azure SQL from file 02), create a table of 200 short FAQ entries with embeddings generated by a Foundry embedding model.
2. Write 20 test questions and the FAQ ID that should answer each.
3. Measure recall@5 with exact search (no index).
4. Add an HNSW (or DiskANN) index and measure recall@5 and latency again.
5. Add a keyword condition or full-text ranking (hybrid) and measure again.
6. Record results in a small table and write two sentences on what changed.

```python
def recall_at_k(results: dict[str, list[str]], expected: dict[str, str], k: int = 5) -> float:
    hits = sum(1 for q, ids in results.items() if expected[q] in ids[:k])
    return hits / len(expected)
```

## Security, cost and reliability notes

- **Security:** vectors can leak information about their source text; protect them like the source data, and apply the same access controls.
- **Security:** always filter by tenant and permission in the database query, never in application code after retrieval.
- **Cost:** vector indexes consume memory and storage; 1 million 1,536-dimension float32 vectors is about 6 GB raw. Quantise or reduce dimensions for large corpora.
- **Cost:** re-embedding is a token cost; only re-embed changed content.
- **Reliability:** index builds on large tables take time and resources; build during low traffic and monitor.

## Common mistakes

- Mixing embeddings from different models in one index.
- Post-filtering after ANN, returning fewer than k results or results from the wrong tenant.
- Adding a separate vector database when the existing database supports vectors.
- Using vector-only search where hybrid would retrieve exact terms (product codes, names) far better.
- Never measuring recall; tuning by gut feel.
- Forgetting to delete vectors when the source is deleted.

## Check yourself

1. What is the difference between exact and approximate nearest neighbour search?
2. Compare HNSW and DiskANN.
3. What is hybrid search and why does it help?
4. Give three reasons to store vectors in Azure AI Search and three to store them in the operational database.
5. Why must you never mix embeddings from different models?
6. How do you measure retrieval quality?
7. Roughly how much raw storage do 1 million 1,536-dimension float32 vectors need?

<details>
<summary>Answers</summary>

1. Exact search compares the query with every vector (perfect recall, linear cost); ANN uses an index to find near neighbours quickly with slightly lower recall.
2. HNSW is an in-memory multi-layer graph: fast and accurate but memory-hungry. DiskANN is a graph index designed to run largely from SSD with high recall at large scale, lower memory, and good filtered search.
3. Combining vector similarity with keyword (BM25) relevance, fused (for example with RRF). It captures both meaning and exact terms such as names and codes.
4. AI Search: document retrieval with indexers and chunking, hybrid plus semantic ranking, permission-aware retrieval across sources. Operational database: transactional consistency with entity data, rich filters and joins, avoiding another service for moderate scale.
5. Different models produce vectors in different spaces; distances between them are meaningless.
6. With a golden set of queries and expected results, measuring recall@k and MRR for each configuration.
7. About 6 GB (1,000,000 × 1,536 × 4 bytes), before index overhead.

</details>

## References

- [Choose an Azure service for vector search](https://learn.microsoft.com/azure/architecture/guide/technology-choices/vector-search)
- [Vector search in Azure AI Search](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Vectors in Azure SQL and SQL Server](https://learn.microsoft.com/sql/relational-databases/vectors/vectors-sql-server)
- [pgvector and DiskANN on PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgdiskann)
- [Vector search in Cosmos DB for NoSQL](https://learn.microsoft.com/azure/cosmos-db/nosql/vector-search)
- [Vector search in Azure DocumentDB](https://learn.microsoft.com/azure/documentdb/vector-search)
- [DiskANN research](https://www.microsoft.com/research/project/project-akupara-approximate-nearest-neighbor-search-for-large-scale-semantic-search/)
- [RAG information retrieval guide](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-information-retrieval)
