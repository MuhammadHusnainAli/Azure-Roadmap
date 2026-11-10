# Embeddings and Vector Similarity

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Embeddings are how AI systems find things by **meaning** rather than by exact words. They power the retrieval step in RAG, semantic caching in the AI gateway, deduplication, clustering, recommendations and agent memory. Choices made here, such as which model, how many dimensions, which distance metric and which index algorithm, directly affect answer quality, latency and cost. Changing them later means re-embedding everything.

## Key concepts

### What an embedding is

An **embedding** is a fixed-length vector of numbers (for example 1,536 floats) that represents the meaning of a piece of content. Similar meanings produce nearby vectors.

```text
"How do I reset my password?"         → [0.012, -0.044, 0.031, ...]
"I forgot my login credentials"        → [0.010, -0.041, 0.029, ...]   ← close: similar meaning
"Quarterly revenue grew 12%"           → [-0.052, 0.008, -0.017, ...]  ← far: different meaning
```

### What embeddings capture, and what they don't

| Captures well | Captures poorly |
|---|---|
| Topic and semantic similarity, paraphrases, synonyms | Exact identifiers (part numbers, SKUs, error codes) |
| Cross-language similarity (multilingual models) | Negation and fine logical distinctions ("refund allowed" vs "refund not allowed" can be close) |
| Rough intent | Numbers, dates and quantities |
| | Recency, authority, permissions (these need metadata and filters) |

This is why production retrieval uses **hybrid search** (vectors + keyword/BM25) plus **filters** and **reranking** (topic 17).

### Distance and similarity metrics

| Metric | Definition | Notes |
|---|---|---|
| **Cosine similarity** | cos θ = (A·B) / (‖A‖ ‖B‖), from −1 to 1 | Most common for text; ignores magnitude |
| **Dot product** | A·B | Equals cosine when vectors are normalised to length 1; fastest |
| **Euclidean (L2) distance** | √Σ(Aᵢ − Bᵢ)² | Smaller is closer; equivalent ranking to cosine for normalised vectors |
| **Manhattan (L1)** | Σ\|Aᵢ − Bᵢ\| | Less common for embeddings |
| **Hamming** | Count of differing bits | For binary-quantised vectors |

OpenAI embedding models return **normalised** vectors (length 1), so cosine similarity and dot product give the same ranking. Use the metric the model's documentation recommends, and configure the same one in your vector index.

```python
import numpy as np
def cosine(a, b): return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

### Dimensionality

| Model (Azure OpenAI, checked 2026-11) | Default dimensions | Max input tokens | Reduce dimensions? |
|---|---|---|---|
| `text-embedding-3-large` | 3,072 | 8,192 | Yes, with the `dimensions` parameter |
| `text-embedding-3-small` | 1,536 | 8,192 | Yes, with `dimensions` |
| `text-embedding-ada-002` (v2) | 1,536 | 8,192 | No |

Other providers in Foundry (for example Cohere embed models) offer their own dimensions, multilingual support and quantised outputs.

**Trade-off:** more dimensions can capture more nuance, but cost more storage, memory and query time. Microsoft reports that `text-embedding-3` models with dimensions reduced below 1,536 still slightly outperform `ada-002` on the MTEB benchmark. Reducing dimensions (for example 3,072 → 1,024) is often a good cost and quality balance. Measure on your data.

You **can't mix** embeddings from different models (or different dimension settings) in one index. Changing the model means re-embedding the corpus.

### Vector indexing algorithms

Comparing a query with every vector (**exhaustive kNN**) is exact but slow at scale. **Approximate nearest neighbour (ANN)** algorithms trade a little recall for big speed gains:

| Algorithm | How it works | Strengths | Where on Azure |
|---|---|---|---|
| **Exhaustive kNN (flat)** | Compare against all vectors | Exact; fine for small sets or ground truth | AI Search (exhaustive KNN), Cosmos DB `flat`, pgvector without index |
| **HNSW** (Hierarchical Navigable Small World) | Multi-layer proximity graph; greedy search from coarse to fine layers | High recall and low latency; tunable (`m`, `efConstruction`, `efSearch`); memory-hungry | Azure AI Search, PostgreSQL pgvector, Azure SQL / Cosmos DB (some modes), Redis |
| **IVF** (inverted file) | Cluster vectors (k-means); search only the nearest clusters (`lists`, `probes`) | Lower memory; needs training on data | pgvector `ivfflat`, Cosmos DB for MongoDB vCore / DocumentDB |
| **DiskANN** | Graph index (Vamana) designed to live mostly on SSD with compressed vectors in memory | Billions of vectors at low cost with high recall; good with filters | **Azure Cosmos DB** (NoSQL), **Azure SQL** vector index, **PostgreSQL** (pg_diskann) |
| **Quantisation** (scalar, binary, product) | Compress vectors (float32 → int8 or bits), often with **rescoring** on full vectors | 4× to 32×+ memory savings | AI Search compression options, Cosmos DB `quantizedFlat`, DiskANN |

Tuning ANN is a recall vs latency vs cost balance. Measure **recall@k** against exhaustive search on a sample.

### Chunking (preview of topic 17)

Embedding models have input limits, and retrieval works best on focused passages. Documents are split into **chunks** (often a few hundred tokens with some overlap, or by headings and layout) before embedding. Chunking strategy often affects RAG quality more than the choice of embedding model.

## How it works on Azure

### Create embeddings (Python, keyless, batched)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI
import numpy as np

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

docs = ["How do I reset my password?", "I forgot my login credentials", "Quarterly revenue grew 12%"]
resp = client.embeddings.create(model="<text-embedding-3-large deployment>", input=docs, dimensions=1024)
vecs = np.array([d.embedding for d in resp.data])

q = client.embeddings.create(model="<text-embedding-3-large deployment>", input=["can't sign in"], dimensions=1024).data[0].embedding
scores = vecs @ np.array(q)            # dot product = cosine for normalised vectors
for doc, s in sorted(zip(docs, scores), key=lambda x: -x[1]):
    print(f"{s:.3f}  {doc}")
```

Send inputs in **batches** (up to 2,048 inputs per request, within the token limits) to reduce overhead, and respect the deployment's TPM quota.

### Where to store vectors on Azure

| Store | When |
|---|---|
| **Azure AI Search** | Default for RAG: hybrid search, semantic ranker, filters, security trimming, integrated vectorisation, Foundry IQ knowledge bases |
| **Azure Cosmos DB** (NoSQL) | Vectors next to operational JSON data (chat history, agent memory, product catalogue) with DiskANN |
| **Azure Database for PostgreSQL** (pgvector, DiskANN) | Relational apps that want vectors in the same database |
| **Azure SQL Database / SQL Server 2025** | Native `VECTOR` type and vector index in SQL apps |
| **Azure Managed Redis** | Low-latency semantic caching and short-lived vectors |
| **Fabric / Databricks** | Analytics-side vector search over lakehouse data |

See topic 09 file 09 for database vector options and topic 17 for AI Search.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Model choice | Start with `text-embedding-3-large` (quality) or `-small` (cost); evaluate a multilingual or domain-specific model if your content needs it. Benchmark retrieval on your own queries, not leaderboards alone |
| Dimensions | Try reduced dimensions (for example 1,024 or 512) and keep them if recall@k on your evaluation set holds |
| Metric | Cosine (or dot product on normalised vectors) for text, matching the model's guidance |
| Index | HNSW for most RAG; DiskANN for very large or cost-sensitive collections; exhaustive for small sets and ground truth |
| Quantisation | Use when memory or cost is the constraint, with rescoring to recover quality |
| Re-embedding plan | Version your embedding model and dimensions in metadata; plan for full re-embedding when you upgrade |

## Hands-on

1. Embed 50 FAQ questions and 10 test queries; compute cosine similarity and inspect the top 3 matches for each query.
2. Repeat with `dimensions=256`, `1024` and `3072`; measure how often the correct FAQ is in the top 3 (recall@3).
3. Test failure modes: queries with exact product codes and with negation. Note where embeddings fail and keyword search would succeed.
4. Load the vectors into an Azure AI Search index with HNSW and compare results with exhaustive kNN.

## Security, cost and reliability notes

- **Security:** embeddings can leak information. Text can sometimes be approximately reconstructed from embeddings, so protect vector stores like the source data, with the same access controls.
- **Security:** store permission metadata (groups, tenant) alongside vectors and filter at query time.
- **Cost:** embedding cost is one-off per chunk plus re-embedding on change; storage and memory cost scale with vector count × dimensions × bytes per dimension.
- **Reliability:** pin the embedding model version; a silent model change makes new vectors incompatible with old ones.

## Common mistakes

- Mixing vectors from different models or dimension settings in one index.
- Relying on vectors alone for IDs, codes and exact terms (use hybrid search).
- Embedding whole documents instead of well-designed chunks.
- Picking the largest dimensions by default without measuring benefit.
- Forgetting to re-embed when changing models.

## Check yourself

1. What does an embedding represent?
2. Why do cosine similarity and dot product rank OpenAI embeddings identically?
3. What are the default dimensions of `text-embedding-3-small` and `-large`?
4. Compare HNSW, IVF and DiskANN.
5. Name two things embeddings capture poorly, and how to compensate.
6. What happens if you switch embedding models?

<details>
<summary>Answers</summary>

1. The meaning of content as a fixed-length numeric vector, where semantically similar content maps to nearby vectors.
2. Because the vectors are normalised to unit length, so the dot product equals cosine similarity.
3. 1,536 for small and 3,072 for large (both reducible with the `dimensions` parameter).
4. HNSW is a layered graph with high recall and low latency but high memory use. IVF clusters vectors and searches the nearest clusters, using less memory but needing training. DiskANN is an SSD-friendly graph index for very large collections at lower cost with high recall.
5. Exact identifiers and codes (compensate with keyword/hybrid search), negation and numbers (compensate with reranking, filters and structured queries), and permissions or recency (use metadata filters).
6. You must re-embed the whole corpus, because vectors from different models aren't comparable.

</details>

## References

- [Understand embeddings in Azure OpenAI](https://learn.microsoft.com/azure/foundry/openai/concepts/understand-embeddings)
- [Embeddings how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/embeddings)
- [Foundry Models sold by Azure: embeddings](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure#embeddings)
- [Vector search in Azure AI Search](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Vector search relevance and scoring (HNSW, exhaustive KNN)](https://learn.microsoft.com/azure/search/vector-search-ranking)
- [Vector indexing in Azure Cosmos DB (DiskANN, quantizedFlat)](https://learn.microsoft.com/azure/cosmos-db/nosql/vector-search)
- [DiskANN paper (Microsoft Research)](https://www.microsoft.com/research/publication/diskann-fast-accurate-billion-point-nearest-neighbor-search-on-a-single-node/)
