# 17 · Retrieval-Augmented Generation and Knowledge

> Tracks: E ● A ● C ○ · Part D · AI on Azure

Retrieval-augmented generation is how AI systems answer from your organisation's knowledge instead of the model's memory: search first, then answer from what was found, with citations and permissions. This topic covers the whole knowledge stack on Azure: RAG fundamentals and failure modes; Azure AI Search tiers, indexes and capacity; ingestion with indexers, skillsets and integrated vectorisation; layout-aware parsing and chunking; vector search, quantisation and filters; hybrid search with RRF and the semantic ranker; agentic retrieval with knowledge sources and knowledge bases; Foundry IQ as the managed knowledge layer for agents; Work IQ, Fabric IQ and web grounding; document-level security trimming; multimodal, structured-data and graph retrieval; evaluation and tuning; and running RAG at scale across tenants and regions.

## What you will be able to do

- Explain the RAG pipeline end to end and diagnose its common failure modes.
- Size and secure an Azure AI Search service, and design an index with the right fields and attributes.
- Build ingestion pipelines with indexers, skillsets, index projections and integrated vectorisation, including deletion tracking.
- Choose parsing and chunking strategies for real documents with headings, tables and images.
- Configure HNSW, filters and vector compression, and measure their effect on recall and size.
- Combine keyword and vector search with RRF, rerank with the semantic ranker, and apply scoring profiles.
- Build agentic retrieval with knowledge sources and knowledge bases, and know what is GA and what is Preview.
- Connect a Foundry IQ knowledge base to a Foundry agent over MCP.
- Decide when to ground on Work IQ, Fabric IQ or the web, and explain the compliance implications.
- Enforce per-user permissions with security filters, native ACLs, SharePoint permissions and Purview labels.
- Handle diagrams, tables, databases and relationship questions with multimodal, text-to-SQL and graph patterns.
- Evaluate retrieval and generation separately with Foundry evaluators and tune one variable at a time.
- Design multi-tenant, multi-region RAG with zero-downtime index changes.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) file 04 (embeddings), [Topic 15](../15-foundry-models-and-azure-openai/README.md) file 09 (embedding models), and [Topic 16](../16-prompt-engineering-and-model-customisation/README.md) files 01 and 03. [Topic 08](../08-storage/README.md) and [Topic 09](../09-databases/README.md) file 09 help with data sources and vector databases.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [RAG Fundamentals](01-rag-fundamentals.md) | 40 min | Python, minimal RAG |
| 02 | [Azure AI Search Fundamentals](02-azure-ai-search-fundamentals.md) | 50 min | Bicep, Python index |
| 03 | [Ingestion: Indexers and Skillsets](03-ingestion-indexers-and-skillsets.md) | 55 min | Portal wizard, REST, Python |
| 04 | [Document Processing and Chunking](04-document-processing-and-chunking.md) | 45 min | Python chunking comparison |
| 05 | [Vector Search](05-vector-search.md) | 50 min | Quantisation recall test |
| 06 | [Hybrid Search and Semantic Ranker](06-hybrid-search-and-semantic-ranker.md) | 50 min | Retrieval mode comparison |
| 07 | [Agentic Retrieval](07-agentic-retrieval.md) | 55 min | Python SDK, knowledge base |
| 08 | [Foundry IQ](08-foundry-iq.md) | 45 min | Foundry agent + MCP |
| 09 | [Work IQ, Fabric IQ and Web Grounding](09-work-iq-fabric-iq-and-web-grounding.md) | 40 min | Web Search tool, grounding map |
| 10 | [Security Trimming and Permissions](10-security-trimming-and-permissions.md) | 50 min | Security filters |
| 11 | [Multimodal and Structured-Data RAG](11-multimodal-and-structured-data-rag.md) | 50 min | Multimodal wizard, SQL tool |
| 12 | [RAG Evaluation and Tuning](12-rag-evaluation-and-tuning.md) | 55 min | Recall@k, Foundry evaluators |
| 13 | [RAG at Scale](13-rag-at-scale.md) | 45 min | Capacity estimate, alias swap |

**AI engineers:** all of it, with 03–07, 10 and 12 in depth. **Architects:** all of it, with 01, 02, 07–10 and 13 in depth. **Consultants:** 01, 08, 09 and 10, plus the cost sections of 02 and 07.

## Completion checklist

- [ ] I can draw the RAG pipeline and say where each failure mode originates.
- [ ] I have created a keyless Azure AI Search service with two replicas and an index with a vector profile and semantic configuration.
- [ ] I have built an indexer pipeline with chunking, embeddings, index projections and deletion detection.
- [ ] I have compared at least two chunking strategies on a golden set.
- [ ] I have measured recall and index size with and without quantisation.
- [ ] I have shown that hybrid plus semantic ranking beats keyword-only and vector-only on my data.
- [ ] I have built a knowledge base and called it through the retrieve action and the MCP endpoint.
- [ ] I have connected a Foundry IQ knowledge base to a Foundry agent with keyless authentication.
- [ ] I can explain the data-boundary implications of web grounding and the licensing of Work IQ.
- [ ] I have implemented and tested security trimming so that one user cannot retrieve another group's documents.
- [ ] I have run Foundry RAG evaluators (groundedness, relevance, retrieval) and tuned one variable at a time.
- [ ] I can choose a multi-tenancy model and perform a zero-downtime index swap with an alias.

## Maps to

- **AI-103**: implement knowledge mining and RAG with Azure AI Search, agentic retrieval and Foundry IQ; evaluate RAG solutions.
- **AZ-305**: design data and AI search solutions, security, capacity and multi-tenancy.
- **AB-100**: position Foundry IQ, Work IQ and Fabric IQ for grounding business agents.
- **AI-901**: RAG concepts (foundational).

## Next

18 · [Agents with Foundry Agent Service](../18-agents-with-foundry-agent-service/README.md)
