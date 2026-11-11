# Foundry IQ

## Why it matters

Every agent needs knowledge, and without a shared layer each team builds its own RAG pipeline: its own index, chunking, permissions logic and refresh jobs. **Foundry IQ** is Microsoft's managed knowledge layer for agents. You build a **knowledge base** once, connect it to many sources (Blob, SharePoint, OneLake, Azure SQL, existing indexes, the web, MCP servers), and many agents reuse it, with permission-aware results and citations. Under the hood it is **agentic retrieval on Azure AI Search** (file 07). For architects it is the "knowledge platform" decision; for consultants it is how to explain grounding to a client who already has Microsoft 365 and Fabric.

## Key concepts

### What Foundry IQ is (and isn't)

| Statement | True? |
|---|---|
| Foundry IQ is built on Azure AI Search agentic retrieval | **Yes.** You need an Azure AI Search service; the knowledge base lives there. |
| You must use Foundry Agent Service to use it | **No.** Knowledge bases can be called from Foundry Agent Service, Microsoft Agent Framework, Copilot Studio, or any app via the Azure AI Search REST API or SDKs. |
| It needs an LLM | **Optional.** Minimal reasoning effort works without one; low and medium effort use an LLM for query planning. |
| It replaces "Azure OpenAI On Your Data" | It is the modern successor pattern: one knowledge base serves many agents and sources, with planning and iterative search. |

### Components

| Component | Description |
|---|---|
| **Knowledge base** | Top-level resource that orchestrates retrieval: knowledge sources, retrieval reasoning effort (minimal, low, medium), retrieval and answer instructions, output mode, optional LLM |
| **Knowledge sources** | Connections to indexed or remote content (file 07 lists kinds) |
| **Agentic retrieval** | Engine that plans, runs parallel sub-queries, reranks and merges |

### Capabilities (from Microsoft Learn)

- One knowledge base connected to **multiple agents**.
- Automated chunking, embedding and metadata extraction for indexed sources, with scheduled incremental refresh.
- Keyword, vector or hybrid queries across indexed and remote sources.
- LLM-based planning, source selection, parallel search and aggregation.
- Extractive results with **citations**.
- ACL synchronisation for supported sources and **Microsoft Purview sensitivity labels**; permissions enforced at query time.
- Queries under the caller's **Entra identity** for end-to-end permission enforcement.
- Connection to **Copilot Studio** agents.

### Sources at a glance

| Source | Mode | Status |
|---|---|---|
| Existing Azure AI Search index | Indexed | GA |
| Azure Blob Storage / ADLS | Indexed (auto pipeline) | GA |
| OneLake | Indexed (auto pipeline) | GA |
| Web (Grounding with Bing) | Remote; requires answer synthesis | GA |
| SharePoint | Indexed (Preview) or remote via the Copilot Retrieval API (Preview; users need a Microsoft 365 Copilot licence) | Preview |
| Azure SQL | Indexed (auto pipeline) | Preview |
| MCP servers | Remote | Preview |
| Work IQ, Fabric Data Agent, Fabric Ontology | Remote | Preview (file 09) |

### Limits by reasoning effort

| Effort | Knowledge sources | Sub-queries | Answer synthesis budget |
|---|---|---|---|
| Minimal | Up to 10 | None (no LLM) | Not supported |
| Low | Up to 3 | Up to 3 | 5,000 tokens |
| Medium | Up to 5 | Up to 5, plus iterative search | 10,000 tokens |

### GA vs Preview

Some Foundry IQ features are GA and others Preview, depending on the Azure AI Search REST API version (2026-04-01 GA; 2026-08-01-preview for planning, answer synthesis and preview sources). The Foundry portal and Azure portal expose **preview-only** experiences.

### The three IQs

| Workload | Layer over | Gives agents |
|---|---|---|
| **Foundry IQ** | Enterprise content across Azure, SharePoint, OneLake, web | Permission-aware knowledge retrieval |
| **Work IQ** | Microsoft 365 (documents, meetings, chats, workflows) | Context on how the organisation works |
| **Fabric IQ** | Microsoft Fabric (ontologies, semantic models, graphs, data agents) | Reasoning over business data and analytics |

Each is standalone; they can be combined (Work IQ and Fabric sources can even be knowledge sources inside a Foundry IQ knowledge base).

## How it works on Azure

### Portal path (Foundry, new portal)

1. **Build → Knowledge**: create or connect a search service that supports agentic retrieval.
2. Create a knowledge base, adding knowledge sources one at a time; set retrieval behaviour.
3. **Build → Agents**: create or select an agent and connect the knowledge base.
4. Test in the playground. Move to code with managed identities before production.

### Code path: connecting a Foundry agent

A Foundry agent reaches a knowledge base through its **MCP endpoint**, using a project connection of category `RemoteTool` with `ProjectManagedIdentity` authentication and audience `https://search.azure.com/`. The only tool supported from Foundry Agent Service is `knowledge_base_retrieve`.

Permissions needed:

| Identity | Role | Scope |
|---|---|---|
| Developer | Foundry User (create agents); Foundry Project Manager (create the connection) | Foundry resource |
| Project managed identity | **Search Index Data Reader** | Search service |
| Search service managed identity | Cognitive Services User | Foundry resource (only if the knowledge base uses an LLM) |

For per-user permission trimming, pass the user's token in the `x-ms-query-source-authorization` header (via structured inputs when using an agent).

### Cost

Foundry IQ inherits the billing of its parts: Azure AI Search (tier plus agentic retrieval tokens beyond the free allowance) and Azure OpenAI tokens for planning and synthesis. Foundry Agent Service doesn't charge for agent instances. For proofs of concept, the Free search tier plus the free agentic retrieval token allowance is enough; the **Serverless (Developer, Preview)** search tier suits bursty or low-volume knowledge workloads.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Foundry IQ vs custom RAG** | Foundry IQ when agents are the consumers and you want shared, governed knowledge with less plumbing. Custom pipelines when you need full control over ranking, unusual data, or GA-only components. |
| **Knowledge base granularity** | One per domain (HR, product, legal) with clear descriptions, rather than one giant base. Agents can use several. |
| **Indexed vs remote SharePoint** | Indexed for control and hybrid search; remote for live permissions and no copies (requires Copilot licences). |
| **Extractive vs answer synthesis** | Microsoft recommends **extractive** for most agent scenarios (the agent reasons over raw content); synthesis for standalone apps and required for web sources. |
| **Reasoning effort** | Minimal for speed and GA; low for most chat; medium only where iterative search measurably helps. |
| **Search tier** | Dedicated S1+ with two or more replicas for production SLA; Serverless (Preview) for experiments. |

## Hands-on

Connect an existing knowledge base (`hr-kb` from file 07) to a Foundry prompt agent.

```python
# pip install "azure-ai-projects>=2.0.0" azure-identity requests
import requests
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

cred = DefaultAzureCredential()
search = "https://<search>.search.windows.net"
mcp_endpoint = f"{search}/knowledgebases/hr-kb/mcp?api-version=2026-08-01-preview"
project_id = "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<acct>/projects/<proj>"

# 1. Project connection (RemoteTool, project managed identity)
arm = get_bearer_token_provider(cred, "https://management.azure.com/.default")
requests.put(
    f"https://management.azure.com{project_id}/connections/hr-kb-mcp?api-version=2025-10-01-preview",
    headers={"Authorization": f"Bearer {arm()}"},
    json={"name": "hr-kb-mcp", "type": "Microsoft.MachineLearningServices/workspaces/connections",
          "properties": {"authType": "ProjectManagedIdentity", "category": "RemoteTool",
                         "target": mcp_endpoint, "isSharedToAll": True,
                         "audience": "https://search.azure.com/", "metadata": {"ApiType": "Azure"}}},
).raise_for_status()

# 2. Agent with the knowledge base as an MCP tool
project = AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>", credential=cred)
agent = project.agents.create_version(
    agent_name="hr-assistant",
    definition=PromptAgentDefinition(
        model="gpt-5-mini",
        instructions=("Use the knowledge base tool to answer every question. "
                      "If the knowledge base doesn't contain the answer, say \"I don't know\". "
                      "Cite the sources you used."),
        tools=[MCPTool(server_label="knowledge-base", server_url=mcp_endpoint,
                       require_approval="never", allowed_tools=["knowledge_base_retrieve"],
                       project_connection_id="hr-kb-mcp")]))

# 3. Ask
oai = project.get_openai_client(agent_name=agent.name)
conv = oai.conversations.create()
r = oai.responses.create(conversation=conv.id, input="How much parental leave do UK staff get?")
print(r.output_text)
```

Before running, grant the project's managed identity **Search Index Data Reader** on the search service.

## Security, cost and reliability notes

- **Permission enforcement only works when configured.** User-level trimming applies only when a source supports document-level permissions *and* synchronisation is configured; otherwise everyone who can call the knowledge base sees everything (file 10).
- **Tool approval.** `require_approval: "never"` is fine for a read-only knowledge tool; keep approvals for tools with side effects.
- **MCP risks.** Microsoft warns that MCP integrations can be susceptible to attacks and cascading failures; only connect trusted servers, especially for MCP knowledge sources.
- **Private networking.** Use private endpoints for the search service; for Copilot Studio, combine Private Link with Power Platform virtual network support.
- **Preview dependencies.** The agent integration uses the preview MCP API version; pin versions and track migration notes.
- **Troubleshooting** (from the FAQ): agent not calling the tool (fix instructions), permissions between agent and knowledge base, wrong project connection, empty knowledge sources.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming Foundry IQ is a separate service with its own SLA | It inherits Azure AI Search and Azure OpenAI SLAs and limits |
| Instructions that don't tell the agent to use the tool | Explicitly require the knowledge tool and citations |
| Forgetting the project managed identity's search role | Assign Search Index Data Reader |
| Using answer synthesis inside agents by default | Prefer extractive data; let the agent compose |
| One knowledge base for every domain | Split by domain with clear descriptions |
| Treating remote SharePoint as licence-free | End users need Microsoft 365 Copilot licences |

## Check yourself

1. What is the relationship between Foundry IQ and agentic retrieval?
2. Is Foundry Agent Service required to use Foundry IQ? Is an LLM?
3. List five knowledge source types and whether each is indexed or remote.
4. What are the source and sub-query limits at low and medium effort?
5. How does a Foundry agent connect to a knowledge base?
6. Which roles are required for that connection to work?
7. When should you use extractive data rather than answer synthesis?
8. How do Foundry IQ, Work IQ and Fabric IQ differ?

<details>
<summary>Answers</summary>

1. Agentic retrieval is the engine; Foundry IQ is the packaged knowledge layer (knowledge bases, sources, integrations with Azure OpenAI and Foundry Agent Service) built on it.
2. No to both: any app can call a knowledge base, and minimal effort works without an LLM (planning needs one).
3. Examples: search index (indexed), Blob (indexed), OneLake (indexed), Azure SQL (indexed, Preview), web (remote), remote SharePoint (remote, Preview), MCP server (remote, Preview), Work IQ (remote, Preview).
4. Low: up to 3 sources and 3 sub-queries; medium: up to 5 sources and 5 sub-queries with iterative search.
5. Through the knowledge base's MCP endpoint, using a RemoteTool project connection with the project managed identity and the `knowledge_base_retrieve` tool.
6. Foundry User and Foundry Project Manager for the developer; Search Index Data Reader for the project managed identity; Cognitive Services User for the search identity if the knowledge base uses an LLM.
7. In most agent scenarios, so the agent reasons over raw content; answer synthesis suits standalone apps and is required for web sources.
8. Foundry IQ covers enterprise content retrieval; Work IQ covers Microsoft 365 work context; Fabric IQ covers business data semantics in Fabric.

</details>

## References

- [What is Foundry IQ?](https://learn.microsoft.com/azure/foundry/agents/concepts/what-is-foundry-iq)
- [Foundry IQ FAQ](https://learn.microsoft.com/azure/foundry/agents/concepts/foundry-iq-faq)
- [Connect a Foundry IQ knowledge base to Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect)
- [Agentic retrieval overview](https://learn.microsoft.com/azure/search/agentic-retrieval-overview)
- [Knowledge sources](https://learn.microsoft.com/azure/search/agentic-knowledge-source-overview)
- [Build smarter agents faster with Foundry IQ (Foundry blog)](https://devblogs.microsoft.com/foundry/build-smarter-agents-faster-with-foundry-iq/)
