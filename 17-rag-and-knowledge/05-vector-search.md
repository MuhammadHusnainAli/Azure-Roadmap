# Vector Search

## Why it matters

Vector search finds content by meaning rather than by matching words, so "time off for remote staff" can find a chunk about "leave entitlement for teleworkers". It is the core of semantic retrieval in RAG. It is also where most capacity and cost problems appear: vectors are large, ANN indexes live in memory, and a careless choice of dimensions or algorithm can force a bigger tier. This file covers how Azure AI Search stores and queries vectors and how to make them smaller without losing relevance.

## Key concepts

### Embedding space

Content and queries are converted to vectors by the **same** embedding model ([Topic 15 file 09](../15-foundry-models-and-azure-openai/09-embedding-models.md)). Similar meanings cluster together. A vector query finds the `k` nearest document vectors to the query vector.

### Algorithms

| Algorithm | Type | Behaviour | Memory (vector quota) |
|---|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | Approximate nearest neighbour (ANN) | Navigates a layered graph; fast, high recall, tunable | Consumes quota (graph in memory) |
| **Exhaustive KNN** | Exact | Brute-force scan of all vectors | Doesn't consume vector index quota |

A field configured for HNSW can also be queried exhaustively with `"exhaustive": true` (useful to build ground truth for measuring ANN recall). A field configured only for exhaustive KNN can't be queried with HNSW.

### HNSW parameters

| Parameter | When | Effect |
|---|---|---|
| `m` | Index time | Neighbours per node; higher → better recall, more memory |
| `efConstruction` | Index time | Candidate list size while building; higher → better graph, slower indexing |
| `efSearch` | Query time | Candidate queue size; higher → better recall, more latency |
| `metric` | Both | `cosine`, `dotProduct`, `euclidean`, `hamming` (binary) |

Azure OpenAI embeddings use **cosine**. For normalised vectors, `dotProduct` gives the same ranking slightly faster.

### Scores

For cosine, `@search.score` is not the raw cosine. It is `1 / (1 + cosine_distance)`, so the range is about 0.333–1.0 and always decreases as similarity worsens. Don't compare raw scores across query types; hybrid results are fused with RRF (file 06).

### Filters with vectors

| Mode | Behaviour | Trade-off |
|---|---|---|
| **`preFilter`** (default for indexes created after about October 2023, recommended) | Filter applied during HNSW traversal | Maximises recall; more CPU for very selective filters |
| **`postFilter`** | Filter applied per shard after traversal | Can miss results (false negatives) with selective filters or small `k` |
| **`strictPostFilter`** (Preview) | Filter applied after the global top-k | Highest risk of false negatives |

Security trimming filters (file 10) should use pre-filtering so users always get a full set of permitted results.

### Making vectors smaller

| Technique | Saving | Notes |
|---|---|---|
| **Scalar quantisation** (int8) | About 4× smaller vector index | Small recall loss; rescoring with originals recovers most of it |
| **Binary quantisation** (1 bit) | Up to about 28× smaller | Best for >1,024 dimensions and zero-centred embeddings; use rescoring |
| **Rescoring with oversampling** | Recovers quality | Query returns `k × oversampling` candidates, then rescores with full-precision vectors (or binary dot product) |
| **Truncated dimensions (MRL)** | Proportional | text-embedding-3 models support Matryoshka truncation, e.g. 3,072 → 1,024 |
| **Narrow types** | 2–4× | `Collection(Edm.Half)` (float16), `Collection(Edm.SByte)` for pre-quantised int8 |
| **`stored: false`** | Removes the retrievable copy on disk | Irreversible; you can't return raw vectors, and partial updates that omit the field lose it |

Microsoft's guidance describes combining these techniques to cut vector costs by up to about 92.5% while measuring relevance with NDCG.

### Multi-vector fields (Preview)

Vector fields can be nested inside complex collections, so one document can hold up to 100 vectors (for example a page with several image and paragraph vectors). Nested vectors are one level deep only, and the semantic ranker doesn't work on nested vectors.

### Query shapes

| Query | Use |
|---|---|
| `VectorizedQuery` | You supply the query vector |
| `VectorizableTextQuery` | Service vectorises text with the index's vectoriser |
| `VectorizableImageUrlQuery` / binary | Multimodal vectorisers (file 11) |
| Multiple vector queries | Query several vector fields (for example title and content); results fused with RRF |
| `weight` | Relative weight of a vector query in fusion |
| `threshold` (Preview) | Minimum similarity to include a result |

## How it works on Azure

Vector configuration lives in the index:

```json
"vectorSearch": {
  "algorithms": [{ "name": "hnsw-1", "kind": "hnsw",
                   "hnswParameters": { "m": 4, "efConstruction": 400, "efSearch": 500, "metric": "cosine" } }],
  "compressions": [{ "name": "sq-1", "kind": "scalarQuantization",
                     "rescoringOptions": { "enableRescoring": true, "defaultOversampling": 4,
                                           "rescoreStorageMethod": "preserveOriginals" } }],
  "vectorizers": [{ "name": "aoai", "kind": "azureOpenAI",
                    "azureOpenAIParameters": { "resourceUri": "https://<foundry>.openai.azure.com",
                                               "deploymentId": "text-embedding-3-large",
                                               "modelName": "text-embedding-3-large" } }],
  "profiles": [{ "name": "default", "algorithm": "hnsw-1", "compression": "sq-1", "vectorizer": "aoai" }]
}
```

The vectoriser authenticates to Azure OpenAI with the search service's managed identity when no key is supplied.

Vector index quota is per partition on Dedicated tiers (for example 35 GB per S1 partition on newer services) and 300 MB per index on Serverless (Preview). Check usage with the index statistics API (`vectorIndexSize`).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **HNSW vs exhaustive** | HNSW for almost everything; exhaustive for small indexes, highly filtered queries, or recall benchmarking |
| **Dimensions** | Larger embeddings aren't always better. Try 3,072 vs 1,024 (truncated) and measure recall@k on your data. |
| **Quantisation** | Default to scalar quantisation with rescoring for large indexes; try binary for high-dimension embeddings and large corpora |
| **`k` value** | Retrieve more candidates (for example 50) than you send to the model, then rerank (file 06) |
| **Vector-only or hybrid** | Vector-only misses exact identifiers, codes and rare names. For RAG, hybrid is the default (file 06). |
| **Database vectors instead** | When vectors must live with transactional data, use Cosmos DB, PostgreSQL (pgvector/DiskANN) or Azure SQL vectors ([Topic 09 file 09](../09-databases/09-vector-search-in-azure-databases.md)) |

## Hands-on

Measure the effect of quantisation on recall.

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizableTextQuery

cred = DefaultAzureCredential()
full = SearchClient("https://<search>.search.windows.net", "docs-float32", cred)
sq   = SearchClient("https://<search>.search.windows.net", "docs-int8", cred)

def ids(client, q, exhaustive=False, oversampling=None):
    vq = VectorizableTextQuery(text=q, k_nearest_neighbors=10, fields="content_vector",
                               exhaustive=exhaustive, oversampling=oversampling)
    return [r["id"] for r in client.search(search_text=None, vector_queries=[vq], top=10, select=["id"])]

questions = [l.strip() for l in open("questions.txt")]
for q in questions:
    truth = set(ids(full, q, exhaustive=True))          # exact neighbours
    hnsw  = set(ids(full, q))
    quant = set(ids(sq, q))
    quant_rescored = set(ids(sq, q, oversampling=4))
    print(len(truth & hnsw)/10, len(truth & quant)/10, len(truth & quant_rescored)/10)
```

Then compare `vectorIndexSize` for both indexes:

```python
from azure.search.documents.indexes import SearchIndexClient
ic = SearchIndexClient("https://<search>.search.windows.net", cred)
for name in ["docs-float32", "docs-int8"]:
    s = ic.get_index_statistics(name)
    print(name, s["vector_index_size"])
```

Write down the recall and size trade-off; that is the evidence for your tier decision.

## Security, cost and reliability notes

- **Vector quota is a hard limit.** Indexing fails once exceeded. Monitor `vectorIndexSize` against quota and alert at 80%.
- **Cost lever.** Quantisation and truncation can avoid adding partitions, which is the biggest vector cost driver on Dedicated tiers.
- **Embeddings can leak content.** Vectors can be partially inverted; protect indexes like the source data (RBAC, private endpoints, `stored: false` if raw vectors aren't needed).
- **Model changes.** Changing the embedding model means re-embedding the whole corpus and rebuilding the index; plan it like a migration (file 13).
- **Older services.** Some indexes created before late 2023 default to `postFilter` and can't use compression; recreate them.

## Common mistakes

| Mistake | Fix |
|---|---|
| Different models for indexing and querying | Use one model; the vectoriser must match the embedding skill |
| `postFilter` with selective security filters | Use `preFilter` |
| Sending all `k` vector hits to the LLM | Over-retrieve, rerank, then trim |
| Ignoring vector quota during sizing | Calculate vectors × dimensions × bytes, plus graph overhead |
| Assuming bigger dimensions are better | Measure; truncation often costs little recall |
| Setting `stored: false` then relying on merge updates | Always send the vector in updates, or keep `stored: true` |

## Check yourself

1. How do HNSW and exhaustive KNN differ in accuracy, speed and vector quota use?
2. What do `m`, `efConstruction` and `efSearch` control?
3. Which similarity metric should you use with Azure OpenAI embeddings?
4. Why is `preFilter` recommended, especially for security filters?
5. How much does scalar and binary quantisation reduce vector index size?
6. What does rescoring with oversampling do?
7. What are the risks of `stored: false`?
8. How would you measure ANN recall in your own index?

<details>
<summary>Answers</summary>

1. HNSW is approximate, fast, and uses quota for its in-memory graph; exhaustive KNN is exact, slower, and doesn't consume vector index quota.
2. `m`: neighbours per node; `efConstruction`: candidate list size during graph build; `efSearch`: candidate queue size at query time.
3. Cosine (or dotProduct for normalised vectors).
4. It applies the filter during traversal so you get a full set of permitted results; post-filtering can return too few or miss results.
5. Scalar (int8) about 4×; binary up to about 28×.
6. Retrieves extra candidates from the compressed index, then reorders them using full-precision vectors (or binary dot products) to recover relevance.
7. It's irreversible, raw vectors can't be returned, and partial updates that omit the vector field lose it.
8. Compare HNSW top-k with exhaustive KNN top-k (`exhaustive: true`) for a set of queries and compute overlap.

</details>

## References

- [Vector search overview](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Relevance in vector search (HNSW, KNN, metrics)](https://learn.microsoft.com/azure/search/vector-search-ranking)
- [Compress vectors with quantisation](https://learn.microsoft.com/azure/search/vector-search-how-to-quantization)
- [Vector query filters](https://learn.microsoft.com/azure/search/vector-search-filters)
- [Multi-vector field support (Preview)](https://learn.microsoft.com/azure/search/vector-search-multi-vector-fields)
- [Vector storage options](https://learn.microsoft.com/azure/search/vector-search-how-to-storage-options)
- [Truncate dimensions (MRL)](https://learn.microsoft.com/azure/search/vector-search-how-to-truncate-dimensions)
