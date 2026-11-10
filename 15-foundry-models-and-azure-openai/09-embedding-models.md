# Embedding Models

## Why it matters

Embedding models turn text (and sometimes images) into vectors for semantic search, RAG retrieval, clustering, deduplication, recommendations and semantic caching. Topic 13 file 04 explained the concepts. This file covers the practical choices on Foundry: which model, how many dimensions, how to batch and stay within quotas, what it costs, and how to avoid re-embedding a corpus because of a careless decision.

## Key concepts

### Embedding models in Foundry (checked 2026-11)

| Model | Provider / category | Dimensions | Max input | Notes |
|---|---|---|---|---|
| `text-embedding-3-large` | Azure OpenAI (sold by Azure) | 3,072 (reducible with `dimensions`) | 8,192 tokens | Highest quality OpenAI embedding; strong multilingual (MIRACL 54.9 vs 31.4 for ada-002) |
| `text-embedding-3-small` | Azure OpenAI | 1,536 (reducible) | 8,192 tokens | Cheaper, good quality (MIRACL 44.0) |
| `text-embedding-ada-002` (v2) | Azure OpenAI | 1,536 (fixed) | 8,192 tokens | Legacy; keep only for existing indexes |
| `Cohere-embed-v5-Pro` / `-Fast`, `embed-v-4-0` | Cohere (sold by Azure) | Model-specific (check card) | Model-specific | Multilingual and multimodal options; quantised output types |
| `Cohere-embed-v3-english` / `-multilingual` | Cohere (partner) | 1,024 | 512 tokens | Older generation |
| Open embedding models (managed compute) | Hugging Face and others | Varies | Varies | Self-hosted option for special domains |

Microsoft notes you **can't upgrade between embedding models**: moving from `ada-002` to `text-embedding-3-large` requires regenerating all embeddings. Embedding models have extended lifecycle timelines compared with chat models, but still check the retirement schedule.

### The `dimensions` parameter

`text-embedding-3` models can return **shortened** vectors via `dimensions` (for example 256, 512, 1,024). Microsoft notes that OpenAI's MTEB testing found these models outperform `ada-002` even when reduced below 1,536 dimensions. Fewer dimensions means less storage, less memory and faster search. Validate the effect on your own recall@k.

### Limits that shape batching

| Limit | Value (checked 2026-11) |
|---|---|
| Max inputs per `/embeddings` request (array) | 2,048 |
| Max tokens per `/embeddings` request (sum across inputs) | 300,000 |
| Max tokens per individual input | 8,192 (for text-embedding-3 and ada-002 v2) |
| Rate limits | TPM and RPM per deployment (quota tiers apply) |

### Cost

Embedding cost = tokens embedded × price per 1,000 (or 1 million) tokens. It's usually small per document, but large corpora and frequent re-embedding add up, and storage and search costs (AI Search vectors, database memory) often exceed the embedding API cost. The main cost levers are:

| Lever | Effect |
|---|---|
| Model choice (small vs large) | `-small` is several times cheaper per token |
| Dimensions | Fewer dimensions means lower vector storage and query cost |
| Chunking | Fewer, better chunks means fewer tokens embedded and less noise |
| Incremental updates | Embed only new or changed chunks (content hashes) |
| Batch deployment | Global Batch supports embeddings for large offline jobs at a discount (check model support) |
| Quantisation in the store | AI Search scalar or binary quantisation, Cosmos DB `quantizedFlat`/DiskANN |

## How it works on Azure

### Batched, rate-limit-aware embedding (Python)

```python
import hashlib, time
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI, RateLimitError

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp, max_retries=5)

DEPLOYMENT, DIMS, BATCH = "text-embedding-3-large", 1024, 256

def embed_chunks(chunks: list[str]) -> list[list[float]]:
    vectors = []
    for i in range(0, len(chunks), BATCH):
        batch = chunks[i:i + BATCH]
        while True:
            try:
                r = client.embeddings.create(model=DEPLOYMENT, input=batch, dimensions=DIMS)
                vectors.extend(d.embedding for d in r.data)     # order matches input order
                break
            except RateLimitError:
                time.sleep(10)                                   # SDK retries first; this is a last-resort backoff
    return vectors

def content_hash(text: str) -> str:                              # skip re-embedding unchanged chunks
    return hashlib.sha256(text.encode()).hexdigest()
```

Store `model`, `dimensions` and `content_hash` with each vector so you can detect stale vectors and plan migrations.

### Integrated vectorisation in Azure AI Search

AI Search can call your Foundry embedding deployment during indexing (**AzureOpenAIEmbedding skill**) and at query time (**vectorizer**), using a managed identity, so you don't write embedding code at all. This is the default path for most RAG ingestion (topic 17).

```json
"vectorizers": [{
  "name": "foundry-embed",
  "kind": "azureOpenAI",
  "azureOpenAIParameters": {
    "resourceUri": "https://<resource>.openai.azure.com",
    "deploymentId": "text-embedding-3-large",
    "modelName": "text-embedding-3-large"
  }
}]
```

The vector field's `dimensions` in the index must match the embedding output (for example 1,024 if you shortened). With managed identity, grant the search service's identity **Foundry User** (or Cognitive Services OpenAI User on a standalone Azure OpenAI resource).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Model | `text-embedding-3-large` for quality and multilingual; `-small` for cost; Cohere for strong multilingual and multimodal retrieval with rerankers; benchmark on your queries |
| Dimensions | Start at 1,024 for `-large` and compare with 3,072; keep the smallest that holds recall |
| Same model for queries and documents | Always; vectors from different models or dimension settings aren't comparable |
| Hosted vs self-hosted | Hosted for most; self-hosted (managed compute) for special domains, air-gapped needs or very high steady volumes |
| Re-embedding plan | Version embeddings; build a blue/green index migration path for model changes |

## Hands-on

1. Embed 1,000 chunks with `-small` and `-large` at 512, 1,024 and full dimensions; measure recall@5 on 50 labelled queries and record cost and index size.
2. Implement content hashing so a second run only embeds changed chunks.
3. Configure an AI Search index with integrated vectorisation using managed identity and query it with text (the vectoriser embeds the query).
4. Hit the rate limit deliberately with large parallel batches and observe 429 handling.
5. Calculate the monthly cost of embedding a 2-million-chunk corpus once, plus 5% daily churn.

## Security, cost and reliability notes

- **Security:** embeddings derive from your data and can leak information; protect vector stores like source data, with access control and private endpoints.
- **Security:** use managed identity between AI Search and Foundry; disable keys.
- **Cost:** the biggest waste is re-embedding unchanged content and oversized dimensions.
- **Reliability:** embedding jobs should be resumable (checkpoint progress) and idempotent (upsert by chunk ID).

## Common mistakes

- Mixing embeddings from different models or dimension settings in one index.
- Exceeding 2,048 inputs or 300,000 tokens per request.
- Re-embedding the full corpus on every pipeline run.
- Forgetting to set the same `dimensions` in the index schema.
- Choosing `ada-002` for new work.

## Check yourself

1. What are the default dimensions of `text-embedding-3-large` and `-small`?
2. What does the `dimensions` parameter do, and which models support it?
3. What are the per-request limits on the embeddings endpoint?
4. Why can't you upgrade an index from ada-002 to text-embedding-3 in place?
5. What's integrated vectorisation in AI Search?
6. Name three ways to cut embedding-related cost.

<details>
<summary>Answers</summary>

1. 3,072 and 1,536.
2. It returns shortened vectors with fewer dimensions; supported by the text-embedding-3 models (not ada-002).
3. Up to 2,048 inputs and 300,000 total tokens per request, with each input up to 8,192 tokens.
4. Vectors from different models aren't comparable; you must re-embed all content with the new model.
5. AI Search calls an embedding deployment during indexing (skill) and at query time (vectoriser), so you don't have to write embedding code.
6. Use the smaller model where quality allows, reduce dimensions, embed only changed chunks (hashing), use batch for large offline jobs, and quantise vectors in the store.

</details>

## References

- [Foundry Models sold by Azure: embeddings](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure#embeddings)
- [Embeddings how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/embeddings)
- [Understand embeddings](https://learn.microsoft.com/azure/foundry-classic/openai/concepts/understand-embeddings)
- [Azure OpenAI quotas and limits](https://learn.microsoft.com/azure/foundry/openai/quotas-limits)
- [Integrated vectorisation in Azure AI Search](https://learn.microsoft.com/azure/search/vector-search-integrated-vectorization)
- [Azure OpenAI vectorizer](https://learn.microsoft.com/azure/search/vector-search-vectorizer-azure-open-ai)
