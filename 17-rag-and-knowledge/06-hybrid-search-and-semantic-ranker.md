# Hybrid Search and Semantic Ranker

## Why it matters

Neither keyword nor vector search is enough on its own. Keyword search (BM25) is precise for product codes, names, error numbers and exact phrases, but misses paraphrases. Vector search captures meaning, but can miss rare terms and identifiers. **Hybrid search** runs both and fuses the results; the **semantic ranker** then reorders the top results with a cross-encoder language model. Microsoft's own benchmarks and guidance consistently show hybrid plus semantic ranking as the strongest classic retrieval configuration for RAG, and semantic ranking is built into agentic retrieval. If you change only one thing in a weak RAG system, this is usually it.

## Key concepts

### Ranking layers in Azure AI Search

```text
L1 retrieval                     fusion                 L2 rerank                 optional
───────────────                  ──────                 ─────────                 ────────
BM25 (keyword) ─┐
                ├──► RRF ──► top 50 ──► semantic ranker ──► @search.rerankerScore ──► scoring profile
vector query(s)─┘                                          captions, answers         (if configured after rerank)
```

### Reciprocal Rank Fusion (RRF)

When two or more queries run in parallel (hybrid, or several vector queries), Azure AI Search fuses results with RRF:

- For each result list, each document gets `1 / (rank + k)`, where `k` is a constant (about 60).
- Scores are summed across lists; documents ranked well in several lists rise to the top.
- RRF uses **ranks, not raw scores**, so BM25's unbounded scores and cosine scores don't need normalising.

Number of parallel executions: one text query plus one vector query on one field = 2; text plus one vector query across two fields = 3.

Controls:

| Parameter | Effect |
|---|---|
| `vectorQueries[].weight` | Relative weight of a vector query in fusion (for example 2 to favour semantics, 0.5 to favour keywords) |
| `hybridSearch.maxTextRecallSize` (Preview) | How many BM25 results feed into fusion; lowering it reduces the keyword leg's influence |
| `k` on the vector query | How many nearest neighbours feed into fusion |
| `top` | How many fused results are returned |

### Semantic ranker

A premium feature (billed by usage, with a free monthly allowance) that uses multilingual deep-learning models adapted from Bing.

| Capability | Description |
|---|---|
| **L2 reranking** | Rescores the top 50 BM25 or RRF results; produces `@search.rerankerScore` from 0 to 4 |
| **Captions and highlights** | Verbatim extracts that summarise each result |
| **Semantic answers** | Optional direct answers for question-like queries |
| **Query rewrite** (Preview) | Generates up to 10 query variants (typo fixes, synonyms) before retrieval |

Inputs: up to 2,048 tokens per document summary, built from the semantic configuration's `title` (128 tokens), `keywords` (128 tokens) and `content` fields **in priority order**. Put the most informative fields first.

Reranker score meanings:

| Score | Meaning |
|---|---|
| 4 | Highly relevant, answers the question completely |
| 3 | Relevant but lacks some detail |
| 2 | Somewhat relevant, partial |
| 1 | Related, answers a small part |
| 0 | Irrelevant |

You can use the reranker score as a threshold (for example drop results below 1.5) but expect slight distribution shifts across infrastructure and model updates.

### Scoring profiles

Scoring profiles boost relevance using business rules. They apply only to **non-vector** fields.

| Element | Example |
|---|---|
| **Text weights** | A match in `title` counts 3× a match in `content` |
| **Freshness** | Boost documents modified in the last 90 days |
| **Magnitude** | Boost by rating, popularity or priority |
| **Distance** | Boost by geographic proximity |
| **Tag** | Boost documents whose tags match values passed in the query (for example user's department) |

By default semantic ranking is the last step; you can configure scoring profiles to apply **after** semantic reranking when business rules must override pure relevance.

### Query rewriting

Two kinds:

| Kind | Where |
|---|---|
| **Semantic query rewrite** (Preview) | `queryRewrites: "generative|count-5"`; the service creates variants and retrieves with all of them. Rewritten queries may drop exact identifiers. |
| **Application-side rewrite** | Your LLM turns a conversational follow-up ("what about part-timers?") into a standalone query using chat history. Agentic retrieval does this automatically (file 07). |

## How it works on Azure

A hybrid semantic query in REST:

```json
POST /indexes/policies/docs/search?api-version=2026-04-01
{
  "search": "leave entitlement part-time staff",
  "vectorQueries": [{ "kind": "text", "text": "leave entitlement part-time staff",
                      "fields": "content_vector", "k": 50, "weight": 1 }],
  "queryType": "semantic",
  "semanticConfiguration": "default",
  "captions": "extractive",
  "answers": "extractive|count-3",
  "top": 5,
  "select": "title,content,source_url"
}
```

Enable semantic ranker on the service (`semanticSearch: free` or `standard`) and define a semantic configuration in the index.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Hybrid by default** | Yes for RAG. Use pure vector only for non-text or cross-lingual cases where keywords add nothing. |
| **Semantic ranker on?** | Yes for RAG over prose. Adds latency (often a few hundred ms) and usage cost; measure. |
| **How many results to send the LLM** | Rerank 50, send top 3–10, optionally filtered by reranker score threshold |
| **Weights** | Tune vector weight only after building a golden set; defaults are a good start |
| **Scoring profiles** | Use for business rules (freshness, authority), not to fix bad chunking |
| **Query rewrite** | Helpful for typo-heavy or terse user queries; risky for exact-match identifiers |
| **Synonym maps** | Cheap fix for domain vocabulary (for example "PTO" ↔ "annual leave") in keyword search |

## Hands-on

Compare four retrieval modes on a golden set (questions with the ID of the chunk that answers them).

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizableTextQuery

client = SearchClient("https://<search>.search.windows.net", "policies", DefaultAzureCredential())

def run(q, mode):
    vq = [VectorizableTextQuery(text=q, k_nearest_neighbors=50, fields="content_vector")]
    kwargs = dict(top=5, select=["id"])
    if mode == "keyword":  r = client.search(search_text=q, **kwargs)
    if mode == "vector":   r = client.search(search_text=None, vector_queries=vq, **kwargs)
    if mode == "hybrid":   r = client.search(search_text=q, vector_queries=vq, **kwargs)
    if mode == "semantic": r = client.search(search_text=q, vector_queries=vq, query_type="semantic",
                                             semantic_configuration_name="default", **kwargs)
    return [d["id"] for d in r]

golden = [("How many leave days do part-timers get?", "policy-12-3"), ...]
for mode in ["keyword", "vector", "hybrid", "semantic"]:
    hits = sum(expected in run(q, mode) for q, expected in golden)
    print(mode, f"recall@5 = {hits/len(golden):.2f}")
```

Then add a scoring profile that boosts `last_modified` within 180 days and see whether outdated policies drop out of the top 5.

## Security, cost and reliability notes

- **Filters still apply.** Security and tenant filters run in L1 retrieval; reranking only reorders permitted results.
- **Cost.** Semantic ranker is billed per 1,000 requests beyond the free allowance; query rewrite adds generative processing. Budget at production QPS.
- **Latency.** Each extra vector field adds a parallel query; semantic ranking adds a model call. Measure p95 at expected load.
- **Region availability.** Semantic ranker and query rewrite are not in every region; check before choosing a region.
- **Determinism.** Reranker score distributions can shift slightly; avoid brittle hard thresholds without monitoring.

## Common mistakes

| Mistake | Fix |
|---|---|
| Vector-only RAG | Add the keyword leg (hybrid) |
| Semantic configuration with long, low-value fields first | Order fields by importance; title and keywords first |
| Expecting scoring profiles to boost vector matches | They apply to non-vector fields only |
| Comparing `@search.score` across query types | Scores have different ranges; use `rerankerScore` or ranks |
| Using query rewrite for part-number lookups | Disable rewrite or route exact-match queries to keyword search |
| Tuning weights without a golden set | Build the golden set first (file 12) |

## Check yourself

1. Why does RRF use ranks rather than raw scores?
2. How many parallel query executions does a text query plus one vector query over two fields create?
3. How many results does the semantic ranker rerank, and what score range does it output?
4. Which fields feed the semantic ranker, and why does order matter?
5. What does query rewrite do, and what is its risk?
6. Which fields can a scoring profile boost?
7. When would you apply a scoring profile after semantic reranking?
8. What does `maxTextRecallSize` control?

<details>
<summary>Answers</summary>

1. BM25 and vector scores have incompatible ranges; ranks can be combined directly.
2. Three.
3. The top 50; `@search.rerankerScore` from 0 to 4.
4. The semantic configuration's title, keywords and content fields; inputs are trimmed to a token budget in priority order, so important fields must come first.
5. It generates query variants (typo fixes, synonyms) to improve recall; rewritten queries may drop exact identifiers.
6. Non-vector fields (text, numeric, date, geo, string collections).
7. When business rules (freshness, authority, tags) must influence the final order beyond pure semantic relevance.
8. How many BM25 results are passed into hybrid fusion.

</details>

## References

- [Hybrid search overview](https://learn.microsoft.com/azure/search/hybrid-search-overview)
- [Relevance scoring with RRF](https://learn.microsoft.com/azure/search/hybrid-search-ranking)
- [Semantic ranking](https://learn.microsoft.com/azure/search/semantic-search-overview)
- [Query rewrite (Preview)](https://learn.microsoft.com/azure/search/semantic-how-to-query-rewrite)
- [Add scoring profiles](https://learn.microsoft.com/azure/search/index-add-scoring-profiles)
- [Create a vector query (weights and hybrid parameters)](https://learn.microsoft.com/azure/search/vector-search-how-to-query)
