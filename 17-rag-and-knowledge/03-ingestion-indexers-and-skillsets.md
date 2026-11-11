# Ingestion: Indexers and Skillsets

## Why it matters

A RAG system can only retrieve what was ingested, in the shape it was ingested. Ingestion decides whether content stays fresh, whether deleted documents disappear, whether tables and images survive, and whether every chunk carries the metadata needed for filtering and permissions. Azure AI Search's **indexers** and **skillsets** give you a low-code ingestion pipeline with change tracking, enrichment and **integrated vectorisation**. Engineers configure and debug it; architects decide between pull (indexers), push (your own pipeline) and Foundry IQ's generated pipelines.

## Key concepts

### Pull vs push

| Model | How | Choose when |
|---|---|---|
| **Pull (indexer)** | The search service crawls a supported data source on demand or on a schedule (as often as every 5 minutes) | Data is in a supported source; you want change tracking and enrichment without code |
| **Push** | Your code sends documents via the index API or SDK | Unsupported sources, sub-5-minute freshness, event-driven updates, custom pipelines |
| **Knowledge source pipelines** | Some agentic retrieval knowledge sources (Blob, ADLS Gen2, OneLake, ingested SharePoint) auto-generate the data source, skillset, index and indexer | Agentic retrieval and Foundry IQ (files 07, 08) |

### Supported indexer data sources

GA: Azure Blob Storage, ADLS Gen2, Azure SQL Database, SQL Managed Instance, SQL Server on Azure VMs, Azure Cosmos DB (NoSQL), Azure Table Storage, Microsoft OneLake.
Preview: Azure Files, Azure Database for MySQL, SharePoint in Microsoft 365, Cosmos DB for MongoDB, Cosmos DB for Gremlin. (Cosmos DB for Cassandra isn't supported.)

### Indexer stages

| Stage | What happens |
|---|---|
| 1. **Document cracking** | Open files and extract text, metadata and optionally images (`imageAction`, extra cost) |
| 2. **Field mappings** | Map source fields to index fields (renames, type fixes, Base64 keys) |
| 3. **Skillset execution** | Optional enrichment: chunking, OCR, layout, embeddings, language, custom skills |
| 4. **Output field mappings** | Map enriched outputs into index fields |

Rules: one indexer reads **one** data source and writes **one** index, but many indexers can write to the same index. A service runs one indexer job per search unit.

### Change and deletion tracking

| Source | Change detection | Deletion detection |
|---|---|---|
| Blob / ADLS Gen2 | Automatic (last-modified timestamp) | Soft delete: native blob soft delete or a custom metadata flag |
| Azure SQL | SQL integrated change tracking or a high-water mark column (`rowversion`) | Integrated change tracking, or a soft-delete column |
| Cosmos DB | `_ts` high-water mark | Soft-delete property |

Without deletion detection, deleted source documents remain searchable indefinitely, which is a serious data-protection risk in RAG.

### Skillsets

A skillset is an array of skills attached to an indexer. Skills read and write an in-memory **enriched document tree** rooted at `/document`. Each skill has a **context** (once per document, or once per item in a collection like `/document/pages/*`), inputs and outputs.

| Skill | Purpose |
|---|---|
| **Text Split** | Chunk by pages (characters or, in preview, tokens) or sentences, with overlap. Free. |
| **Document Layout** | Document Intelligence layout model; Markdown or text output by section; text and image extraction |
| **Azure Content Understanding** | Extraction and semantic chunking; tables and figures as Markdown; cross-page tables; optional AI figure descriptions (Preview). GA in REST API 2026-04-01. |
| **Azure OpenAI Embedding** | Vectors from text-embedding-3-small/-large or ada-002 |
| **Azure Vision multimodal embeddings** | Image and text vectors (Preview) |
| **AML skill / Foundry catalogue** | Embeddings from catalogue models |
| **GenAI Prompt skill** | Call a chat model during indexing, for example to verbalise images or summarise |
| **OCR, Image Analysis** | Text and tags from images |
| **Language skills** | Entities, key phrases, language detection, PII, translation |
| **Custom Web API** | Your own code (for example an Azure Function) |

Built-in skills that call Foundry Tools need an attached **billable Foundry resource** beyond 20 documents per indexer per day (Content Understanding has no free allowance).

### Integrated vectorisation

Integrated vectorisation adds vector encoding **during indexing** (an embedding skill in the skillset) and **during queries** (a **vectoriser** defined in the index). The vectoriser must use the same embedding model as the skill:

| Embedding skill | Matching vectoriser |
|---|---|
| AzureOpenAIEmbedding | Azure OpenAI vectoriser |
| Custom Web API skill | Custom Web API vectoriser |
| Azure Vision multimodal | Azure Vision vectoriser |
| AML skill (Foundry catalogue) | Foundry model catalogue vectoriser (Preview) |

### Index projections (one-to-many)

By default an indexer creates one search document per source document. For RAG you want **one search document per chunk**. **Index projections** map each chunk in `/document/pages/*` to its own document in the index, with a parent key linking back to the source and parent fields (title, URL, permissions) copied onto every chunk.

## How it works on Azure

| Tool | Use |
|---|---|
| **Import data (new) wizard** in the portal | Generates data source, skillset with chunking and embedding, index with vectoriser, and indexer |
| REST API / SDKs (`azure-search-documents`) | Full control, source-controlled definitions |
| **Debug sessions** | Inspect the enrichment tree for a single document and fix skills interactively (not in Serverless preview) |
| **Indexer status and execution history** | Errors, warnings, items processed |
| **Managed identity** | Search service identity reads sources (Storage Blob Data Reader) and calls Foundry (Cognitive Services OpenAI User) |
| **Shared private link** | Indexer outbound access to private resources |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Indexer or push** | Indexers for supported sources and minute-level freshness; push for real-time or complex transformations. Many teams mix them. |
| **Text Split vs Layout vs Content Understanding** | Text Split for clean text; Content Understanding or Document Layout for PDFs with headings, tables and figures (file 04) |
| **Where to embed** | Integrated vectorisation keeps index and query encoding consistent; external pipelines give more control over models and batching |
| **Schedule** | Match business freshness needs; every run costs indexer time and enrichment calls |
| **Embedding throughput** | Large initial loads can hit Azure OpenAI rate limits (429s); use dedicated embedding deployments with enough quota |
| **Enrichment cache** | Incremental enrichment caches skill outputs in Storage so unchanged documents aren't re-enriched after a skillset edit |

## Hands-on

Use the portal wizard first, then reproduce it as code.

1. Upload 20 PDFs to a Blob container.
2. Grant the search service's managed identity **Storage Blob Data Reader** on the storage account and **Cognitive Services OpenAI User** on the Foundry resource.
3. In the search service, open **Import data (new)** → Azure Blob Storage → **RAG** scenario. Choose your embedding deployment and authenticate with the managed identity.
4. Inspect what was created: data source, skillset (Text Split + Azure OpenAI Embedding + index projections), index (with vectoriser) and indexer.
5. Export the definitions:

```bash
TOKEN=$(az account get-access-token --resource https://search.azure.com --query accessToken -o tsv)
for t in datasources skillsets indexers indexes; do
  curl -s -H "Authorization: Bearer $TOKEN" \
    "https://<search>.search.windows.net/$t?api-version=2026-04-01" > $t.json
done
```

6. Delete a blob, enable soft delete in the data source definition, re-run the indexer and confirm the chunk disappears from search.

Python skeleton for running and monitoring an indexer:

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents.indexes import SearchIndexerClient

client = SearchIndexerClient("https://<search>.search.windows.net", DefaultAzureCredential())
client.run_indexer("policies-indexer")
status = client.get_indexer_status("policies-indexer")
print(status.last_result.status, status.last_result.item_count, status.last_result.failed_item_count)
for err in status.last_result.errors[:5]:
    print(err.key, err.error_message)
```

## Security, cost and reliability notes

- **Use managed identities** for data source and skill connections; avoid connection strings with keys.
- **Carry permissions through ingestion.** ACLs or group IDs must be indexed with every chunk (file 10); Blob/ADLS and SharePoint indexers can ingest permission metadata.
- **Private networking.** Use shared private links for indexer outbound connections; skillset-based private outbound access needs S2 or higher.
- **Cost.** Image extraction, Document Layout, Content Understanding and embedding calls are billed per document or token; a full re-index of a large corpus can be expensive. Use enrichment caching and avoid unnecessary resets.
- **Reliability.** Indexers have maximum run times; large loads resume on the next run. Watch for throttling from embedding endpoints and retry.

## Common mistakes

| Mistake | Fix |
|---|---|
| No deletion detection | Configure soft delete for every source |
| One search document per PDF | Use index projections to index chunks |
| Different embedding model at index and query time | Vectoriser must match the embedding skill |
| Running indexers on the Free tier for real data | Use a billable tier; Free times out on enrichment |
| Editing a skillset and resetting everything | Use incremental enrichment and targeted resets |
| Ignoring indexer warnings | Monitor execution history; truncated or skipped documents are silent failures |

## Check yourself

1. What is the difference between pull and push ingestion?
2. List the four indexer stages.
3. How does a Blob indexer detect deleted files?
4. What is the enriched document tree, and what is a skill context?
5. What does integrated vectorisation add at index time and at query time?
6. Why are index projections important for RAG?
7. When is a billable Foundry resource required for built-in skills?
8. Which identity roles does the search service need to index blobs and embed with Azure OpenAI?

<details>
<summary>Answers</summary>

1. Pull: an indexer crawls a supported source on a schedule; push: your code sends documents to the index API.
2. Document cracking, field mappings, skillset execution, output field mappings.
3. Through soft-delete detection: native blob soft delete or a custom metadata soft-delete column.
4. An in-memory tree rooted at `/document` that skills read from and write to; the context sets how often a skill runs and where outputs attach.
5. An embedding skill vectorises chunks during indexing; a vectoriser in the index converts query text to a vector at query time.
6. They create one search document per chunk, with parent metadata copied, so retrieval returns focused chunks rather than whole documents.
7. Beyond 20 documents per indexer per day for most built-in AI skills; always for the Content Understanding skill.
8. Storage Blob Data Reader on the storage account and Cognitive Services OpenAI User on the Foundry/Azure OpenAI resource.

</details>

## References

- [Indexers in Azure AI Search](https://learn.microsoft.com/azure/search/search-indexer-overview)
- [Skillset concepts](https://learn.microsoft.com/azure/search/cognitive-search-working-with-skillsets)
- [Integrated vectorisation](https://learn.microsoft.com/azure/search/vector-search-integrated-vectorization)
- [Define index projections](https://learn.microsoft.com/azure/search/search-how-to-define-index-projections)
- [Change and deletion detection for blobs](https://learn.microsoft.com/azure/search/search-how-to-index-azure-blob-changed-deleted)
- [Azure Content Understanding skill](https://learn.microsoft.com/azure/search/cognitive-search-skill-content-understanding)
