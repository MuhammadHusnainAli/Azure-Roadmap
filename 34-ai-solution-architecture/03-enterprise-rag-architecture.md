# Enterprise RAG Architecture

## Why it matters

A RAG demo over 50 PDFs takes an afternoon. Enterprise RAG over millions of documents from SharePoint, file shares, databases and line-of-business systems, with per-user permissions, daily changes, multiple languages, tables and images, audit requirements and quality targets, is an architecture problem. Retrieval quality determines answer quality, permission enforcement determines whether the system is safe, and ingestion design determines freshness and cost. [Topic 17](../17-rag-and-knowledge/README.md) covers each RAG technique in depth; this file assembles them into an end-to-end enterprise architecture using Azure AI Search, Foundry IQ, Agent Service and the Architecture Center's RAG design and evaluation guide.

## Key concepts

### Two pipelines

| Pipeline | Steps | Azure services |
|---|---|---|
| **Data (ingestion) pipeline** | Pull/push sources → parse (layout, tables, images) → **chunk** → **enrich** (titles, summaries, keywords, entities, ACLs) → **embed** → **persist** to index; incremental updates and deletions | AI Search indexers and skillsets, integrated vectorisation, Document Intelligence/Content Understanding, Functions/Container Apps for custom steps, Data Factory/Fabric pipelines, Event Grid for change events |
| **Application (query) flow** | User query → orchestrator/agent decides retrieval (query rewriting, decomposition) → search (keyword, vector, hybrid, semantic ranking, filters including security) → top-N context → model generates grounded answer with citations | Foundry Agent Service, Agent Framework, AI Search (hybrid + semantic ranker), agentic retrieval / **Foundry IQ** knowledge bases, Foundry models |

### Standard vs agentic RAG

| | Standard RAG | Agentic RAG |
|---|---|---|
| Retrieval | Single query to one index | Model plans subqueries, chooses sources, iterates |
| Sources | One or few indexes | Multiple knowledge sources (indexes, SharePoint, OneLake, web) |
| Latency/cost | Lower | Higher |
| Fit | Focused Q&A | Complex questions across sources |
| Azure | AI Search query from app | Agentic retrieval / Foundry IQ knowledge bases, agent tools ([Topic 17 files 07–08](../17-rag-and-knowledge/07-agentic-retrieval.md)) |

### Enterprise concerns

| Concern | Design |
|---|---|
| **Permissions** | Index ACL/label metadata; filter by user identity (security trimming) or document-level access control features; on-behalf-of retrieval for SharePoint ([Topic 17 file 10](../17-rag-and-knowledge/10-security-trimming-and-permissions.md)) |
| **Freshness** | Incremental indexing schedules, change notifications, deletion handling; freshness SLOs per source |
| **Scale** | Index partitioning by domain, multiple indexes vs one, search service tiers/replicas/partitions, embedding throughput ([Topic 17 file 13](../17-rag-and-knowledge/13-rag-at-scale.md)) |
| **Multimodal and structured data** | Tables, images, diagrams; text-to-SQL for structured data ([Topic 17 file 11](../17-rag-and-knowledge/11-multimodal-and-structured-data-rag.md)) |
| **Quality** | Evaluation of retrieval (recall, precision) and generation (groundedness, relevance) per step |
| **Governance** | Data classification, Purview labels, residency, retention of indexes and logs |
| **Multilingual** | Language analysers, multilingual embeddings, translation strategy |
| **Citations and trust** | Return source links; show confidence; human review for critical answers |

### Design and evaluation method (Architecture Center guide)

Determine the solution domain and requirements → gather representative documents and test queries → choose chunking strategy → decide enrichment → choose embedding model → configure the index → choose search types (vector, full-text, hybrid, multiple searches) → evaluate each step → consider agentic RAG.

## How it works on Azure

Reference architecture:

```text
Sources: SharePoint, file shares, Blob/ADLS, SQL, ServiceNow/Confluence (connectors)
  ↓ (indexers / Fabric pipelines / custom Functions on change events)
Processing: Document Intelligence layout → chunking (structure-aware) → enrichment (summaries, keywords, ACLs, sensitivity label)
           → embeddings (text-embedding-3-large or similar via private endpoint) → AI Search indexes (by domain)
Knowledge layer: Foundry IQ knowledge base over indexes + SharePoint + OneLake sources
Application: Chat UI → app (auth, conversation ownership) → Foundry agent (knowledge tool, other tools) → models via gateway
Security: Entra auth, security filters by user groups/OIDs, Purview, private endpoints, guardrails (indirect attack detection)
Ops: evaluation datasets per domain, continuous evaluation, tracing, index freshness monitoring, cost dashboards
```

Security-trimmed query (REST-style filter on group IDs from the user's token):

```json
{
  "search": "parental leave policy Germany",
  "vectorQueries": [{ "kind": "text", "text": "parental leave policy Germany", "fields": "contentVector", "k": 50 }],
  "queryType": "semantic",
  "semanticConfiguration": "default",
  "filter": "group_ids/any(g: search.in(g, 'a1b2..., c3d4...'))",
  "top": 8,
  "select": "title, chunk, url, sensitivity_label"
}
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **One index vs many** | Separate indexes by domain/permission model/language for relevance and manageability; knowledge bases federate them. |
| **Chunking** | Structure-aware chunks with overlap; evaluate sizes; keep parent document context. |
| **Hybrid + semantic ranker** | Default for enterprise text; pure vector rarely best alone. |
| **Agentic retrieval** | For complex, multi-source questions; budget latency and cost. |
| **Push vs pull indexing** | Indexers for supported sources; push via custom pipelines for complex logic. |
| **Permissions model** | Prefer native document-level access/OBO where available; otherwise sync ACLs into the index with frequent updates. |

## Hands-on

1. Build an ingestion pipeline with layout-aware chunking and ACL enrichment into AI Search.
2. Implement security-trimmed hybrid queries and test with two users.
3. Create a Foundry IQ knowledge base over two indexes and a SharePoint source; compare with direct queries.
4. Create a 100-question evaluation set and measure retrieval recall and groundedness (Topic 17 file 12).
5. Measure index freshness (time from document change to searchable).

## Security, cost and reliability notes

- **Oversharing**: RAG exposes whatever the index contains; enforce permissions and clean source permissions (Topic 23 file 05).
- **Indirect prompt injection** via documents: enable document attack detection and treat content as untrusted.
- **Cost**: embeddings at ingestion, search tier and replicas, semantic ranker queries, model tokens for long contexts.
- **Reliability**: search replicas across zones, reindex capability, monitoring of indexer failures.

## Common mistakes

| Mistake | Fix |
|---|---|
| No permission filtering | Security trimming |
| Fixed-size chunking without evaluation | Structure-aware, evaluated chunking |
| One giant index for everything | Domain indexes and knowledge bases |
| No deletion handling | Track deletes in ingestion |
| Judging quality by demos | Evaluation datasets and metrics |

## Check yourself

1. Describe the data pipeline and query flow.
2. Compare standard and agentic RAG.
3. How do you enforce permissions in enterprise RAG?
4. List six enterprise RAG concerns.
5. When should you split indexes?
6. What are the steps of the RAG design and evaluation method?

<details>
<summary>Answers</summary>

1. Data: ingest, parse, chunk, enrich, embed, persist with incremental updates. Query: orchestrator plans retrieval, runs searches with filters, packages top-N context, model generates grounded answers with citations.
2. Standard: single query to an index, cheaper and faster; agentic: model plans subqueries across sources iteratively, better for complex questions but costlier.
3. Index ACL metadata with security filters or document-level access features, OBO retrieval for SharePoint, and clean source permissions.
4. Any six: permissions, freshness, scale, multimodal/structured data, quality evaluation, governance, multilingual, citations/trust.
5. By domain, permission model, language, update frequency or scale needs.
6. Define domain and requirements, gather documents and queries, choose chunking, enrichment, embedding model, index configuration and search types, evaluate each step, consider agentic RAG.

</details>

## References

- [RAG solution design and evaluation guide](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [RAG preparation phase](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-preparation-phase)
- [Agentic retrieval in Azure AI Search](https://learn.microsoft.com/azure/search/search-agentic-retrieval-concept)
- [What is Foundry IQ?](https://learn.microsoft.com/azure/foundry/agents/concepts/what-is-foundry-iq)
- [Security trimming in Azure AI Search](https://learn.microsoft.com/azure/search/search-security-trimming-for-azure-search)
- [Document-level access control](https://learn.microsoft.com/azure/search/search-document-level-access-overview)
