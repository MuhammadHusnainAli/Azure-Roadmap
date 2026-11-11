# RAG Fundamentals

## Why it matters

Language models know a lot about the world and nothing about your organisation. They also don't know what changed yesterday, can't cite their sources, and don't respect your permissions. **Retrieval-augmented generation (RAG)** fixes this by retrieving relevant content from your own sources at query time and giving it to the model as grounding. It is the most common enterprise generative AI pattern, and it is also where most AI projects fail quietly: answers that sound right but are built on the wrong chunk. Engineers build the pipeline, architects choose its components and security model, and consultants qualify whether a client's content is ready for it.

## Key concepts

### The idea in one line

Instead of asking the model to remember, you **search**, then **ask the model to answer from what you found**, and **cite** it.

### The RAG pipeline end to end

```text
INGESTION (offline, continuous)                      QUERY (online, per request)
──────────────────────────────                       ──────────────────────────────
sources (SharePoint, Blob, SQL, OneLake, web)        user question (+ conversation)
   │ extract text, layout, images                        │ query rewrite / planning
   ▼                                                     ▼
parse ─► chunk ─► enrich ─► embed ─► index  ◄────── retrieve (keyword + vector)
                                                         │ rerank (semantic ranker)
                                                         ▼
                                                    top chunks + citations
                                                         │ prompt with grounding
                                                         ▼
                                                    LLM generates answer
                                                         │
                                                    evaluate (groundedness, relevance)
```

| Stage | Purpose | Covered in |
|---|---|---|
| Ingest and parse | Get clean text, tables and images out of documents | Files 03, 04 |
| Chunk | Split content into retrievable units | File 04 |
| Enrich and embed | Add metadata and vectors | Files 03, 05 |
| Index | Store for fast keyword and vector retrieval | Files 02, 05 |
| Retrieve | Hybrid search, filters, permissions | Files 05, 06, 10 |
| Rerank | Reorder by semantic relevance | File 06 |
| Plan (agentic) | Break complex questions into sub-queries | Files 07, 08 |
| Generate | Answer with citations from the grounding data | [Topic 16](../16-prompt-engineering-and-model-customisation/README.md) |
| Evaluate | Measure retrieval and answer quality | File 12 |

### Why not just use a long context window?

Even million-token windows don't make RAG obsolete:

| Issue | Why retrieval still wins |
|---|---|
| Corpus size | Enterprise corpora are far larger than any window |
| Cost and latency | Sending everything on every call is slow and expensive |
| Quality | Models get worse at finding facts buried in long, noisy context |
| Permissions | You must filter content per user before the model sees it |
| Freshness | The index updates continuously; prompts don't |

### Microsoft's list of RAG challenges

Azure AI Search documentation frames RAG around five challenges: **query understanding** (vague or conversational questions that don't match document wording), **multi-source data access**, **token constraints**, **response time** (answers in seconds) and **security and governance** (users only see what they're allowed to see).

### Two RAG approaches in Azure AI Search

| | **Classic RAG** | **Agentic retrieval** |
|---|---|---|
| How | App sends one query (hybrid + semantic ranking), then passes results to an LLM | An LLM plans sub-queries using conversation history, runs them in parallel across knowledge sources, merges and reranks, returns structured grounding with citations |
| Objects | Index, indexer, skillset | Knowledge sources and a knowledge base, called through the retrieve action |
| Status | GA features | Query planning in Preview |
| Choose when | You need GA only, simplicity and speed, existing orchestration, or fine control | Client is an agent or chatbot, questions are complex or conversational, highest relevance needed, new builds |

Microsoft's guidance is to **start new RAG implementations with agentic retrieval** and consider migrating existing ones. **Foundry IQ** (file 08) packages agentic retrieval as a knowledge layer for Foundry agents.

### Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| "I don't know" when the answer exists | Retrieval miss: chunking, vocabulary mismatch, keyword-only search | Hybrid search, better chunks, query rewriting |
| Confident wrong answer | Wrong chunk retrieved, or model ignored grounding | Semantic ranker, stricter prompt, groundedness evaluation |
| Answer from the wrong version of a document | Stale or duplicate content | Freshness, deduplication, version metadata and filters |
| Missing table or figure data | Text-only parsing | Layout-aware parsing, multimodal (files 04, 11) |
| Partial answer to a multi-part question | Single query can't cover all parts | Agentic retrieval sub-queries |
| User sees content they shouldn't | No security trimming | Document-level access control (file 10) |
| Slow answers | Too many chunks, sequential calls, large models | Fewer, better chunks; parallel retrieval; smaller generator |

## How it works on Azure

| Layer | Azure options |
|---|---|
| Retrieval engine | **Azure AI Search** (primary), vector search in Cosmos DB, PostgreSQL, Azure SQL ([Topic 09 file 09](../09-databases/09-vector-search-in-azure-databases.md)) |
| Knowledge layer for agents | **Foundry IQ** (knowledge bases on Azure AI Search), Work IQ, Fabric IQ (file 09) |
| Agent tools | Azure AI Search tool, file search, Foundry IQ knowledge in Foundry Agent Service ([Topic 18 file 05](../18-agents-with-foundry-agent-service/05-built-in-tools.md)) |
| Parsing | Document Intelligence layout, Content Understanding ([Topic 20](../20-foundry-tools/README.md)) |
| Embeddings | Azure OpenAI and other embedding models ([Topic 15 file 09](../15-foundry-models-and-azure-openai/09-embedding-models.md)) |
| Generation | Any Foundry chat or reasoning model |
| Evaluation | Foundry RAG evaluators (file 12) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Agentic retrieval or classic** | Agentic for agents and complex questions; classic for GA-only, simple, low-latency lookups |
| **Azure AI Search or database vectors** | AI Search for documents, hybrid and semantic ranking, many sources, security trimming; database vectors when data and vectors must live with operational records |
| **Build vs Foundry IQ** | Foundry IQ for agents in Foundry with less plumbing; build your own pipeline for full control |
| **How many chunks to send** | Usually 3–10 after reranking; measure, don't guess |
| **Generator model** | Smaller models are often enough once retrieval is good; reasoning models help for synthesis across many sources |
| **Citations** | Always require them; they make errors visible and enable groundedness checks |

## Hands-on

Minimal classic RAG with keyless auth, assuming an index named `policies` with `content`, `title` and a vector field, plus a semantic configuration.

```python
# pip install azure-search-documents openai azure-identity
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizableTextQuery
from openai import OpenAI

cred = DefaultAzureCredential()
search = SearchClient("https://<search>.search.windows.net", "policies", cred)
llm = OpenAI(base_url="https://<foundry>.openai.azure.com/openai/v1/",
             api_key=get_bearer_token_provider(cred, "https://ai.azure.com/.default"))

question = "How many days of leave do part-time staff get?"
results = search.search(
    search_text=question,                                        # keyword
    vector_queries=[VectorizableTextQuery(text=question, k_nearest_neighbors=50,
                                          fields="content_vector")],  # vector (integrated vectoriser)
    query_type="semantic", semantic_configuration_name="default",    # rerank
    top=5, select=["title", "content"],
)
sources = "\n\n".join(f"[{r['title']}]\n{r['content']}" for r in results)

answer = llm.responses.create(
    model="gpt-5-mini",
    instructions="Answer only from SOURCES. Cite titles in brackets. If not found, say you don't know.",
    input=f"SOURCES:\n{sources}\n\nQUESTION: {question}",
)
print(answer.output_text)
```

Then deliberately break it: switch to keyword-only search, then vector-only, then remove the semantic ranker. Note which questions fail in each case. That is the intuition behind files 05 and 06.

## Security, cost and reliability notes

- **Security trimming is not optional** for internal content. Filter by user identity at retrieval time (file 10). Never rely on the prompt to hide content.
- **Indirect prompt injection.** Retrieved documents can contain instructions. Delimit them as data and use Prompt Shields ([Topic 20 file 09](../20-foundry-tools/09-azure-ai-content-safety.md)).
- **Cost drivers:** search service tier and replicas, embedding generation during ingestion, semantic ranker or agentic retrieval usage, and generator input tokens (chunk count × chunk size).
- **Reliability:** use at least two replicas for read availability SLAs and plan index rebuilds without downtime (file 13).

## Common mistakes

| Mistake | Fix |
|---|---|
| Vector-only search for everything | Use hybrid; keywords catch IDs, codes, names |
| One huge chunk per document | Chunk to retrievable units with overlap |
| Judging RAG by reading a few answers | Build a golden dataset and measure retrieval and groundedness |
| No citations | Require them; evaluate them |
| Ignoring stale and duplicate documents | Track versions and freshness |
| Fixing retrieval problems with prompt tweaks | Diagnose: was the right chunk in the top results? |

## Check yourself

1. Describe the RAG pipeline from ingestion to answer.
2. Give three reasons RAG is still needed with very long context windows.
3. What are Microsoft's five RAG challenges?
4. How do classic RAG and agentic retrieval differ?
5. Which approach does Microsoft recommend for new implementations, and what is its status?
6. A user asks a two-part question and gets half an answer. What is the likely cause and fix?
7. Why must permission filtering happen at retrieval, not in the prompt?
8. Name four cost drivers of a RAG solution.

<details>
<summary>Answers</summary>

1. Sources → parse → chunk → enrich and embed → index; then query → (rewrite or plan) → hybrid retrieve → rerank → top chunks with citations → LLM answer → evaluation.
2. Corpus size exceeds the window; cost and latency; quality degrades in long noisy context; permissions; freshness.
3. Query understanding, multi-source data access, token constraints, response time, security and governance.
4. Classic sends one query and the app orchestrates the LLM; agentic retrieval uses an LLM to plan parallel sub-queries across knowledge sources and returns structured grounding with citations.
5. Agentic retrieval; query planning is in Preview.
6. A single query can't cover both parts; use agentic retrieval (query decomposition) or multiple queries.
7. The model can be manipulated or can leak anything in its context; content a user can't access must never reach the prompt.
8. Search tier and replicas, embedding generation, semantic ranker or agentic retrieval usage, generator tokens.

</details>

## References

- [RAG in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
- [What is Azure AI Search?](https://learn.microsoft.com/azure/search/search-what-is-azure-search)
- [Agentic retrieval overview](https://learn.microsoft.com/azure/search/agentic-retrieval-overview)
- [Design and develop a RAG solution (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo)
