# Agentic Retrieval

## Why it matters

Classic RAG sends one query and hopes it finds everything. Real questions are messier: "Compare our parental leave in the UK and Germany and tell me what changed this year" needs several searches, across several sources, informed by the conversation so far. **Agentic retrieval** in Azure AI Search uses an LLM to plan sub-queries, runs them in parallel across **knowledge sources**, reranks and merges the results, and returns structured grounding with citations (and optionally a synthesised answer). It is the engine behind **Foundry IQ** (file 08) and is Microsoft's recommended starting point for new RAG builds. Engineers must understand its objects and costs; architects must understand what is GA and what is Preview.

## Key concepts

### The pipeline

```text
app / agent ──► knowledge base.retrieve(messages or intents)
                    │
                    ├─ 1. query planning (LLM, low/medium effort)  → focused sub-queries per source
                    ├─ 2. parallel execution across knowledge sources (keyword, vector or hybrid)
                    ├─ 3. semantic reranking of each sub-query's results
                    └─ 4. result synthesis → merged content + references + activity log
                                             (optional: answer synthesis, Preview)
```

At **minimal** reasoning effort, the planning step is skipped and the query goes directly to the knowledge sources.

### Objects

| Object | Role |
|---|---|
| **Knowledge source** | A top-level resource defining content. **Indexed** (backed by a search index) or **remote** (queried live via the platform's API). |
| **Knowledge base** | Orchestrates retrieval: references knowledge sources, an optional LLM, retrieval instructions, output mode and reasoning effort |
| **Search index** | Backs indexed knowledge sources; needs a semantic configuration |
| **Semantic ranker** | Used internally for L2 reranking |
| **LLM** (Azure OpenAI in Foundry Models) | Query planning, source selection, web summarisation, answer synthesis |

Knowledge sources must exist before the knowledge base, and both must live on the same search service.

### Knowledge source kinds

| Kind | Indexed / remote | Status |
|---|---|---|
| Search index (wrap an existing index) | Indexed | GA |
| Azure Blob (auto-generated indexer pipeline) | Indexed | GA |
| OneLake (lakehouse, auto-generated pipeline) | Indexed | GA |
| Web (Microsoft Bing) | Remote | GA |
| Azure SQL | Indexed | Preview |
| File (direct upload) | Indexed | Preview |
| Indexed SharePoint | Indexed | Preview |
| Remote SharePoint | Remote | Preview |
| Fabric Data Agent | Remote | Preview |
| Fabric Ontology | Remote | Preview |
| MCP server | Remote | Preview |
| Work IQ | Remote | Preview |

Auto-generated pipelines create the data source, skillset (chunking and optional vectorisation), index and indexer for you.

### Retrieval reasoning effort

| Effort | LLM use | When |
|---|---|---|
| **minimal** | None; queries go straight to sources | Speed and cost matter; simple questions; GA path |
| **low** (default on the knowledge base) | Plans sub-queries and selects sources | Most chat scenarios |
| **medium** | More planning and iterative search | Complex, multi-part questions |

Source selection at low and medium effort uses the knowledge source **name**, the index **description**, and the **retrieval instructions** you write on the knowledge base. Set `alwaysQuery: true` on a source to include it regardless.

### Output modes

| Mode | Returns |
|---|---|
| **Extractive data** | Merged grounding content plus references; your own model writes the answer |
| **Answer synthesis** (Preview) | An LLM-formulated, citation-backed answer, guided by `answerInstructions` |

### GA vs Preview: which API version?

| API version | Use for |
|---|---|
| **2026-04-01** (GA) | Production with GA knowledge source types, **minimal** effort, extractive retrieval |
| **2026-08-01-preview** | Query planning, answer synthesis, low/medium effort, multi-turn messages, preview knowledge sources |

The Azure portal and Foundry portal use preview schemas, so objects created there may need migration to the GA API version.

### MCP endpoint

Every knowledge base is also an **MCP server** at `https://<search>.search.windows.net/knowledgebases/<kb>/mcp?api-version=...`, exposing a `knowledge_base_retrieve` tool. Any MCP client (Foundry Agent Service, GitHub Copilot, Claude, Cursor) can call it. Authenticate with a bearer token whose identity has **Search Index Data Reader**.

## How it works on Azure

### Billing

| Service | Charges for |
|---|---|
| Azure AI Search | **Retrieval tokens** for sub-query execution and semantic ranking (free monthly allowance, then pay-as-you-go standard plan) |
| Azure OpenAI | Input and output tokens for query planning and answer synthesis |

Classic retrieval is billed per query (semantic ranker); agentic retrieval is billed per token and varies with reasoning effort. Microsoft's worked example: 2,000 retrievals with three sub-queries each, reranking 50 chunks of 500 tokens per sub-query, cost about $4.32 in total. Use the response's activity log to see which queries ran against which sources.

### Supported planning models

Knowledge bases accept specific Azure OpenAI deployments (GPT-5 family; GPT-4o and GPT-4.1 variants are listed as deprecated). Assign the search service's managed identity **Cognitive Services OpenAI User** on the Azure OpenAI resource.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Agentic vs classic** | Agentic for agents, multi-part and conversational questions; classic for low latency, GA-only, simple lookups |
| **Reasoning effort** | Start at minimal for GA and speed; use low or medium only where evaluation shows better answers |
| **Number of knowledge sources** | Fewer is cheaper and faster; consolidate where you can. More sources increase fan-out and token use. |
| **Retrieval instructions** | Write them like a prompt: which source for which topic, what to skip |
| **Extractive vs answer synthesis** | Extractive when your agent writes the answer and applies its own policies; synthesis for simple Q&A endpoints |
| **Index vs remote sources** | Indexed for control, citations URLs and security trimming; remote for live data you can't or shouldn't copy |
| **Direct API vs MCP** | REST/SDK in your own app; MCP to plug the same knowledge into any agent framework |

## Hands-on

Create a knowledge base over an existing index and call it (Python SDK, keyless).

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndexKnowledgeSource, SearchIndexKnowledgeSourceParameters,
    KnowledgeBase, KnowledgeSourceReference)
from azure.search.documents.knowledgebases import KnowledgeBaseRetrievalClient
from azure.search.documents.knowledgebases.models import (
    KnowledgeBaseRetrievalRequest, KnowledgeBaseMessage, KnowledgeBaseMessageTextContent,
    SearchIndexKnowledgeSourceParams)

endpoint = "https://<search>.search.windows.net"
cred = DefaultAzureCredential()
ic = SearchIndexClient(endpoint, cred)

ic.create_or_update_knowledge_source(SearchIndexKnowledgeSource(
    name="policies-ks",
    description="HR policies: leave, benefits, conduct, expenses.",
    search_index_parameters=SearchIndexKnowledgeSourceParameters(search_index_name="policies")))

ic.create_or_update_knowledge_base(KnowledgeBase(
    name="hr-kb",
    description="Answers employee HR questions.",
    knowledge_sources=[KnowledgeSourceReference(name="policies-ks")]))

kb = KnowledgeBaseRetrievalClient(endpoint=endpoint, knowledge_base_name="hr-kb", credential=cred)
result = kb.retrieve(KnowledgeBaseRetrievalRequest(
    messages=[KnowledgeBaseMessage(role="user", content=[KnowledgeBaseMessageTextContent(
        text="How much parental leave do UK staff get, and does it differ for part-timers?")])],
    knowledge_source_params=[SearchIndexKnowledgeSourceParams(knowledge_source_name="policies-ks")]))

print(result.response[0].content[0].text)      # merged grounding (JSON with ref_ids)
for ref in result.references or []:
    print(ref.id, ref.reranker_score)
for entry in result.activity or []:            # what ran, against which source
    print(entry)
```

Then:

1. Pass the grounding to a chat model with an instruction to cite `ref_id`s.
2. Switch to a preview API version, attach a GPT-5 deployment, set reasoning effort to low, and ask a two-part question. Compare the activity log (number of sub-queries) and the answer quality.
3. Connect the knowledge base's MCP endpoint to a Foundry agent as an MCP tool ([Topic 18 file 07](../18-agents-with-foundry-agent-service/07-mcp-tools-toolboxes-and-skills.md)).

Class names follow the current `azure-search-documents` documentation; check the SDK changelog for the version you install, as preview and GA shapes differ.

## Security, cost and reliability notes

- **Permissions.** Agentic retrieval supports knowledge-source-level access control, inherits SharePoint permissions for remote SharePoint, honours Entra ID permission metadata indexed from Azure Storage, and supports per-user enforcement at retrieve time (file 10).
- **Sensitivity labels** (Preview) can be ingested for Blob, OneLake and indexed SharePoint sources; map them to every chunk via index projections.
- **Web source.** Bing grounding sends query text outside your tenant; review data-handling terms before enabling it.
- **Cost.** Token-based billing scales with sub-queries, reranked chunk size and reasoning effort. Keep chunks lean and sources few.
- **Latency.** Planning adds an LLM round-trip; minimal effort avoids it.
- **Preview risk.** Pin API versions, and expect schema changes between preview versions.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building production on portal-created preview objects | Recreate with the GA API version where possible |
| Vague knowledge source names and descriptions | The planner selects sources by name, description and instructions; make them precise |
| Medium effort everywhere | Measure; minimal or low is often enough |
| Too many overlapping knowledge sources | Consolidate to reduce fan-out and cost |
| Ignoring the activity log | Use it to debug which sub-queries ran and why |
| Forgetting the planner model's RBAC | Grant the search identity Cognitive Services OpenAI User |

## Check yourself

1. Describe the four stages of agentic retrieval.
2. What is the difference between indexed and remote knowledge sources? Give two examples of each.
3. What happens at minimal retrieval reasoning effort?
4. Which API version is for production, and what does it support?
5. What inputs does the planner use to pick knowledge sources?
6. How is agentic retrieval billed compared with classic semantic ranking?
7. How can an agent built outside Foundry use a knowledge base?
8. What role must an identity have to call the MCP endpoint?

<details>
<summary>Answers</summary>

1. Query planning (LLM generates sub-queries), parallel execution across knowledge sources, semantic reranking, and result synthesis (merged content, references, activity log, optional answer).
2. Indexed sources have a backing search index (search index, Blob, OneLake, Azure SQL); remote sources are queried live (web/Bing, remote SharePoint, MCP server, Work IQ, Fabric Data Agent).
3. LLM query planning is skipped and queries are sent directly to the knowledge sources.
4. 2026-04-01: GA knowledge source types with minimal effort and extractive retrieval.
5. The knowledge source name, the index description, and the retrieval instructions.
6. Agentic retrieval is token-based (search retrieval tokens plus Azure OpenAI planning tokens); classic semantic ranking is query-based.
7. Through the knowledge base's MCP endpoint, which exposes a `knowledge_base_retrieve` tool.
8. Search Index Data Reader on the search service.

</details>

## References

- [Agentic retrieval overview](https://learn.microsoft.com/azure/search/agentic-retrieval-overview)
- [What is a knowledge source?](https://learn.microsoft.com/azure/search/agentic-knowledge-source-overview)
- [Create a knowledge base](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-create-knowledge-base)
- [Query a knowledge base with retrieve or MCP](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-retrieve)
- [Quickstart: agentic retrieval](https://learn.microsoft.com/azure/search/search-get-started-agentic-retrieval)
- [RAG in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
