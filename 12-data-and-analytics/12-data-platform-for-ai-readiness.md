# Data Platform for AI Readiness

> Topic: 12 · Data and Analytics · Tracks: E ● A ● C ● · Status checked: 2026-11

## Why it matters

Most failed AI projects are data projects in disguise. A RAG assistant gives outdated answers because the index refreshes monthly. An agent leaks salaries because the source library had "Everyone" permissions. A fine-tuned model learns from mislabelled examples. **AI readiness** is the discipline of making data fit for a specific AI use: findable, high quality, fresh, permissioned, documented and legally usable. Consultants assess it during qualification (topic 13, file 09); architects design the platform to deliver it; engineers build the pipelines.

## Key concepts

### Readiness differs by AI pattern

| Pattern | Data it needs | What "ready" means |
|---|---|---|
| **RAG over documents** | Policies, manuals, contracts, tickets, wiki pages | Current versions only, de-duplicated, extractable text (OCR for scans), meaningful structure (headings, tables), **source permissions preserved**, metadata for filtering (department, date, product) |
| **Agents over structured data** | Tables, KPIs, semantic models, APIs | Clear business names and descriptions, star schemas, trusted measures, row-level security, low-latency access, read vs write separation |
| **Fine-tuning / distillation** | Input–output example pairs | Representative, correctly labelled, de-identified, licensed for training, enough volume, held-out evaluation set |
| **Evaluation** | Ground-truth Q&A sets, expected outputs | Realistic, covering edge cases, versioned, reviewed by domain experts |
| **Classic ML** | Feature tables with labels | Point-in-time correct features, no leakage, stable definitions |

### The readiness dimensions

| Dimension | Questions to ask | Azure / Fabric capabilities |
|---|---|---|
| **Availability and discoverability** | Do we know where the data is? Can we reach it without copying it everywhere? | Purview Unified Catalog, OneLake catalog, shortcuts, mirroring |
| **Quality** | Accurate, complete, consistent, de-duplicated? | Purview data quality, Lakeflow expectations, notebook tests, Great Expectations-style checks |
| **Freshness** | How stale can an answer be? How fast do changes propagate? | Mirroring, Eventstreams, AI Search indexer schedules, change-driven ingestion (Event Grid) |
| **Access control** | Will the AI respect who may see what? | Entra ID, document-level security trimming in AI Search, OBO identity for Fabric data agents, RLS/OLS, OneLake security, sensitivity labels |
| **Privacy and compliance** | PII present? Lawful basis for this use? Residency? | Purview classifications and labels, DSPM for AI, masking, Presidio-style redaction, region choices |
| **Semantics and context** | Will a model understand the data's meaning? | Descriptions, glossary, semantic models, ontology (Fabric IQ), metadata |
| **Lineage and provenance** | Can we trace an answer to its source? | Purview lineage, citations in RAG responses, document version metadata |
| **Format and structure** | Is content machine-readable and well chunked? | Content Understanding / Document Intelligence, layout-aware chunking, AI Search skills |

### Permissions: the most common blocker

RAG and agents must **never widen access**. A user must only receive answers from content they could open themselves.

| Source | Approach |
|---|---|
| SharePoint / OneDrive | Index with permission metadata and filter at query time (AI Search security trimming with Entra ACLs, or use the Microsoft 365 Copilot / Work IQ path that respects permissions natively) |
| ADLS Gen2 | AI Search indexers can ingest ACLs and RBAC scope from ADLS Gen2 and enforce them at query time (check current support) |
| Databases and Fabric | Query live with the user's identity (OBO): Fabric data agents, RLS on SQL and semantic models |
| Mixed / custom | Store allowed group IDs per chunk and filter with the user's group claims; refresh ACLs when permissions change |

Also clean up oversharing at the source first: "Everyone except external users" links, stale sites, broken inheritance. Purview DSPM for AI and SharePoint Advanced Management help find it.

### Freshness design

| Requirement | Pattern |
|---|---|
| Daily is fine | Scheduled indexer or pipeline |
| Minutes | Event-driven ingestion (Event Grid on blob changes, change feed, mirroring) and incremental indexing |
| Seconds / live state | Don't index: let the agent query the system of record at answer time (tool or data agent) |
| Deletions | Propagate deletes (soft-delete markers, deletion detection policies) so removed documents stop appearing in answers |

## How it works on Azure

### A reference "AI-ready" data platform

```text
Sources ──(mirroring / shortcuts / pipelines / Eventstreams)──► OneLake medallion
   │                                                         │
   │                                   silver: cleansed, PII-tagged, quality-scored
   │                                   gold: data products, semantic models, ontology
   │                                         │
   ├── documents ──► Content Understanding ──► chunk + embed ──► Azure AI Search (Foundry IQ knowledge base, ACL-trimmed)
   │                                                                     │
   │                                                     Foundry agents ─┤
   │                                                                     ├─ Fabric data agent (OBO, RLS)
   │                                                                     └─ tools / APIs for live state
Governance across everything: Purview catalogue, labels, quality, lineage, DSPM for AI, audit
```

### An AI data readiness checklist (per use case)

```text
[ ] Use case and the exact questions it must answer are written down
[ ] Authoritative sources identified, with owners
[ ] Content is current (versioning, archive policy) and duplicates removed
[ ] Text extraction quality checked on a sample (tables, scans, images)
[ ] Sensitive data classified; PII handling decided (exclude, mask, or allow with controls)
[ ] Permissions model decided and tested with real users of different roles
[ ] Freshness target set and ingestion designed to meet it, including deletions
[ ] Metadata available for filtering (date, department, product, region)
[ ] Structured data has descriptions, measures and RLS; semantic model reviewed
[ ] Evaluation set (50–200 questions with expected answers and sources) created with domain experts
[ ] Legal check: licensing and lawful basis for using the data with AI, residency of processing
[ ] Owners agree to maintain sources (governance, not one-off clean-up)
```

### Measuring readiness

| Metric | How |
|---|---|
| Coverage | % of evaluation questions whose answer exists in indexed sources |
| Retrieval quality | Recall@k / precision of the right chunks for evaluation questions (topic 17) |
| Freshness lag | Time from source change to searchable or queryable |
| Permission correctness | Test matrix: users × documents → expected visible or not |
| Data quality score | Purview or pipeline quality rules on key fields |
| Groundedness of answers | AI evaluators in Foundry (topic 22) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Index vs query live | Index unstructured content and slowly changing reference text; query live for transactional state, numbers and anything permission-heavy |
| Clean at source vs clean in pipeline | Fix ownership and permissions at the source; do format and structure clean-up in the pipeline |
| One big index vs many | Separate indexes or knowledge sources by security boundary and domain; one giant index makes trimming and relevance harder |
| Build vs use Microsoft 365 Copilot | If the content is mainly Microsoft 365 documents for internal staff, Microsoft 365 Copilot or Work IQ already respects permissions; build custom RAG when you need custom UX, external users, other sources or control (topic 24) |
| Data for fine-tuning | Prefer RAG and prompting first; fine-tune only with clean, licensed, representative examples and a held-out evaluation set (topic 16) |

## Hands-on

1. Pick a real or sample use case ("HR policy assistant") and complete the checklist above, noting gaps.
2. Take 20 source documents and test text extraction (Content Understanding or Document Intelligence). Record failure types: tables, scans, headers and footers.
3. Build a small AI Search index with a `group_ids` field per chunk and test queries with filters for two different users.
4. Write 30 evaluation questions with expected answers and source documents, and measure how many are answerable from the indexed content.
5. Run a Purview scan over the sources and list sensitive data found. Decide exclude, mask or allow for each type.

## Security, cost and reliability notes

- **Security:** permission trimming must be enforced server-side with the user's verified identity, never by trusting a group list sent by the client.
- **Security:** re-check permissions regularly. ACLs in an index go stale when source permissions change.
- **Cost:** cleaning data up front is cheaper than tokens spent retrieving duplicates and noise. Smaller, curated corpora often give better answers.
- **Reliability:** monitor ingestion lag and failures; a silently failing indexer is how assistants start giving outdated answers.
- **Compliance:** record which data sources each AI system uses (an AI system inventory) for audits and impact assessments (topic 23).

## Common mistakes

- "Index the whole SharePoint" without owners, clean-up or permission trimming.
- Using exported, stale copies of tables for agents instead of live governed queries.
- No deletion handling, so retracted documents keep being cited.
- Ignoring scanned PDFs and complex tables until users complain.
- Building an evaluation set after launch instead of before.
- Assuming that because data is in Azure, it's legally usable for AI training.

## Check yourself

1. How do readiness requirements differ between RAG and fine-tuning?
2. What are three ways to ensure an assistant respects document permissions?
3. When should an agent query live data instead of an index?
4. How do you measure whether your content can answer the target questions?
5. What role does Purview play in AI readiness?
6. Why is deletion propagation important?

<details>
<summary>Answers</summary>

1. RAG needs current, well-structured, permissioned, de-duplicated content with metadata. Fine-tuning needs representative, correctly labelled, de-identified and licensed example pairs with a held-out evaluation set.
2. Security trimming in AI Search using ACL or group fields filtered by the user's verified identity; querying live with the user's identity (OBO) as Fabric data agents do; using Microsoft 365 Copilot or Work IQ, which respect Microsoft 365 permissions natively (plus fixing oversharing at source).
3. For transactional state, numbers and KPIs, permission-heavy data, or anything that must be current to the second.
4. Build an evaluation set of realistic questions with expected answers and sources, then measure coverage, retrieval recall and answer groundedness.
5. It discovers and classifies data, applies labels, scores quality, shows lineage, curates data products, assesses AI data security posture and audits AI interactions.
6. Otherwise deleted or retracted content remains in indexes and keeps appearing in answers, causing wrong or non-compliant responses.

</details>

## References

- [Data management for AI (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/plan#prepare-your-data)
- [RAG solution design and evaluation guide](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Document-level access control in Azure AI Search](https://learn.microsoft.com/azure/search/search-document-level-access-overview)
- [Microsoft Purview DSPM for AI](https://learn.microsoft.com/purview/ai-microsoft-purview)
- [What is Azure Content Understanding?](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Fabric data agent concepts](https://learn.microsoft.com/fabric/data-science/concept-data-agent)
- [Data quality in Purview Unified Catalog](https://learn.microsoft.com/purview/unified-catalog-data-quality)
