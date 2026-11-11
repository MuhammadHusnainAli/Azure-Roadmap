# Third-Party Frameworks on Foundry

## Why it matters

Many teams already have agents built with **LangChain/LangGraph**, **CrewAI**, **LlamaIndex**, the **OpenAI Agents SDK** or the **Claude Agent SDK**, or prefer them for their ecosystems. Foundry doesn't force a rewrite: it offers Foundry models through standard APIs, integration packages (notably `langchain-azure-ai`), framework-agnostic **hosted agents**, **toolboxes** over MCP, and OpenTelemetry-based tracing for popular frameworks. Architects need to know what each path gives and costs; engineers need the integration points so third-party agents still get Foundry identity, governance and observability.

## Key concepts

### Integration layers

| Layer | What Foundry provides | Framework-agnostic? |
|---|---|---|
| **Models** | Foundry Models via OpenAI-compatible v1 API (Responses, Chat Completions), Anthropic SDK for Claude, project endpoint | Yes |
| **Tools** | **Toolbox** MCP endpoint (web search, file search, AI Search, OpenAPI, MCP, skills) | Yes (any MCP client) |
| **Knowledge** | Foundry IQ knowledge bases via MCP or REST | Yes |
| **Hosting** | **Hosted agents**: Responses or Invocations protocol, sessions, identity, scaling | Yes (protocol SDKs) |
| **Safety** | Content filters on models; framework middleware (for example LangChain middleware for Foundry content safety) | Partly |
| **Observability** | OpenTelemetry traces into Application Insights; Foundry tracing UI | Yes, with instrumentation |
| **Evaluation** | Foundry evaluators on traces and datasets | Yes |
| **Memory** | Foundry memory stores via APIs and framework integrations | Partly |

### Hosting paths (from Microsoft's hosted-agent samples)

| Path | Use when | How |
|---|---|---|
| **Agent Framework** | New agents; also supports patterns from AutoGen and Semantic Kernel | `ResponsesHostServer` from `agent_framework_foundry_hosting` |
| **LangGraph** | Existing LangChain/LangGraph code | `langchain_azure_ai.agents.hosting.ResponsesHostServer(graph)` |
| **Bring your own** (CrewAI, OpenAI Agents SDK, Claude Agent SDK, GitHub Copilot SDK, custom) | Existing agents in other frameworks, deployed unchanged | Protocol SDKs `azure-ai-agentserver-responses` / `azure-ai-agentserver-invocations` (recommended) or the lower-level `azure-ai-agentserver-core` adapter |

Official bring-your-own samples include OpenAI Agents SDK (Responses), Claude Agent SDK and GitHub Copilot (Invocations), AG-UI, and LangGraph variants with toolbox and user identity.

### LangChain and LangGraph: `langchain-azure-ai`

| Namespace | Use |
|---|---|
| `langchain_azure_ai.chat_models` | Azure OpenAI and catalogue chat models |
| `langchain_azure_ai.embeddings` | Catalogue embedding models |
| `langchain_azure_ai.vectorstores` / `.retrievers` | Azure AI Search and Cosmos DB vector integrations |
| `langchain_azure_ai.agents` | Use Foundry Agent Service agents as nodes in LangGraph flows |
| `langchain_azure_ai.agents.hosting` | Host compiled LangGraph agents with Responses or Invocations |
| `langchain_azure_ai.agents.middleware` | Foundry content safety and moderation middleware |
| `langchain_azure_ai.tools` | Document Intelligence, Vision, health text analytics, Logic Apps, **Foundry Toolbox** |
| `langchain_azure_ai.chat_history` | Persisted and memory-backed chat history |
| `langchain_azure_ai.callbacks` | Run events and OpenTelemetry traces |

Extras: `[tools]`, `[opentelemetry]`, `[hosting]`. Use the default package for the new Foundry (`azure-ai-projects>=2.0`); `langchain-azure-ai[v1]` for Foundry classic. Python 3.11+.

### Other frameworks

| Framework | Foundry integration |
|---|---|
| **OpenAI Agents SDK** | Point at Foundry's OpenAI-compatible endpoint with Entra tokens; host as bring-your-own (Responses); OpenTelemetry instrumentation for Foundry tracing |
| **Claude Agent SDK** | Use Claude models deployed in Foundry (Anthropic SDK against the Foundry endpoint, [Topic 15 file 03](../15-foundry-models-and-azure-openai/03-other-model-families.md)); host as bring-your-own (Invocations sample) |
| **CrewAI** | Use Foundry models through OpenAI-compatible or LiteLLM-style configuration; containerise and host with the protocol SDKs |
| **LlamaIndex** | Use Foundry models and Azure AI Search vector store integrations; host as bring-your-own |
| **Semantic Kernel / AutoGen** | Migrate to Agent Framework (file 03); SK tracing is supported meanwhile |

### Tracing

Foundry's framework tracing (Preview) configures:

- Automatic tracing for **Agent Framework** and **Semantic Kernel**.
- The **Microsoft OpenTelemetry distro** (`microsoft-opentelemetry`) for **LangChain/LangGraph**.
- OpenTelemetry instrumentation for the **OpenAI Agents SDK**.

Traces land in Application Insights connected to the project and appear in Foundry's tracing views, enabling evaluation on traces.

## How it works on Azure

Host an existing LangGraph agent (official pattern):

```python
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langchain_azure_ai.agents.hosting import ResponsesHostServer

def build_chat_model() -> ChatOpenAI:
    credential = DefaultAzureCredential()
    project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=credential)
    return ChatOpenAI(
        model=os.environ.get("AZURE_AI_MODEL_DEPLOYMENT_NAME", "gpt-5-mini"),
        base_url=str(project.get_openai_client().base_url),
        api_key=get_bearer_token_provider(credential, "https://ai.azure.com/.default"))

graph = create_agent(build_chat_model(), tools=[])
ResponsesHostServer(graph).run(port=int(os.environ.get("PORT", "8088")))
```

Deploy with `azd` as a hosted agent ([Topic 18 file 04](../18-agents-with-foundry-agent-service/04-hosted-agents.md)); the agent gets an Entra identity, endpoint and sessions.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Keep or migrate a framework** | Keep LangGraph and others if the team is productive and the code is solid; host them on Foundry. Migrate SK/AutoGen to Agent Framework. |
| **Integration depth** | Minimum: Foundry models + tracing. Better: + toolbox for tools and identity. Best: hosted agent for endpoint, identity, publishing and governance. |
| **Protocol SDKs vs core adapter** | Protocol SDKs to stay aligned with the platform contract; core adapter only when you need low-level control. |
| **Model access** | Use Entra token providers with OpenAI-compatible clients; avoid API keys. |
| **Safety parity** | Ensure content filtering and prompt-injection defences apply regardless of framework (model-side filters, middleware). |
| **Vendor lock-in** | Third-party frameworks keep code portable; Foundry hosting adds platform features. Toolboxes over MCP keep tools portable too. |

## Hands-on

1. Take a small LangGraph agent and replace its model with a Foundry deployment via `ChatOpenAI` + Entra token provider (above).
2. Add Foundry tools through a **toolbox** using `langchain_azure_ai.tools` and call web search from the graph.
3. Install `microsoft-opentelemetry` and enable tracing; view the run in Foundry.
4. Wrap with `ResponsesHostServer` and deploy as a hosted agent; call it via `project.get_openai_client(agent_name=...)`.
5. Compare with the OpenAI Agents SDK bring-your-own sample: what's similar (Responses protocol, identity), what differs (framework code).

## Security, cost and reliability notes

- **Third-party code risk**: Microsoft's hosted-agent terms note that third-party frameworks, servers and non-Azure models are used at your own risk; review data flows and licences.
- **Identity**: use the hosted agent's Entra identity and toolbox-managed auth instead of embedding credentials in framework config.
- **Telemetry**: without instrumentation, third-party agents are black boxes; make tracing mandatory.
- **Dependency hygiene**: pin framework versions; set `PYTHONNOUSERSITE=1` for hosted Python agents as Microsoft recommends.
- **Cost**: frameworks differ in how many model calls they make per task; measure tokens per task in traces.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rewriting a working LangGraph agent just to use Foundry | Host it with `langchain-azure-ai[hosting]` |
| API keys in framework model configs | Entra token providers |
| No tracing for third-party agents | Microsoft OpenTelemetry distro / instrumentation |
| Custom tool auth inside each framework | Toolbox with central auth |
| Using `langchain-azure-ai[v1]` with the new Foundry | Default package targets `azure-ai-projects>=2.0` |
| Hand-rolling the hosted agent HTTP contract | Use the protocol SDKs |

## Check yourself

1. Which Foundry capabilities are framework-agnostic?
2. What are the three hosting paths in Microsoft's hosted-agent samples?
3. Name five namespaces in `langchain-azure-ai` and their purpose.
4. How do you host a compiled LangGraph graph on Foundry?
5. Which packages implement the hosted-agent contract for bring-your-own frameworks?
6. How does Foundry trace LangChain and OpenAI Agents SDK runs?
7. When should you migrate rather than host a third-party framework?
8. How should a third-party framework authenticate to Foundry models?

<details>
<summary>Answers</summary>

1. Models via OpenAI-compatible APIs, toolboxes over MCP, Foundry IQ, hosted agents via protocol SDKs, OpenTelemetry tracing and evaluation.
2. Agent Framework, LangGraph (via `langchain-azure-ai`), and bring-your-own (CrewAI, OpenAI Agents SDK, Claude Agent SDK, custom) via protocol SDKs.
3. Any five: `chat_models`, `embeddings`, `vectorstores`, `retrievers`, `agents`, `agents.hosting`, `agents.middleware`, `tools`, `chat_history`, `callbacks`.
4. Wrap it with `langchain_azure_ai.agents.hosting.ResponsesHostServer(graph).run()` and deploy as a hosted agent with azd.
5. `azure-ai-agentserver-responses` and `azure-ai-agentserver-invocations` (or the lower-level `azure-ai-agentserver-core`).
6. LangChain/LangGraph via the Microsoft OpenTelemetry distro; OpenAI Agents SDK via OpenTelemetry instrumentation; traces go to Application Insights and Foundry.
7. When the framework is deprecated or archived (Semantic Kernel features, AutoGen) or when you need Agent Framework-specific features.
8. With Entra ID token providers (scope `https://ai.azure.com/.default`) against the Foundry OpenAI-compatible endpoint.

</details>

## References

- [Get started with LangChain and LangGraph in Foundry](https://learn.microsoft.com/azure/foundry/how-to/develop/langchain)
- [Host LangGraph agents as Foundry hosted agents](https://learn.microsoft.com/azure/foundry/how-to/develop/langchain-hosted-agents)
- [Configure tracing for agent frameworks (Preview)](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-framework)
- [Hosted agent samples (Agent Framework, LangGraph, bring-your-own)](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)
- [What are hosted agents?](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents)
- [langchain-azure-ai on GitHub](https://github.com/langchain-ai/langchain-azure)
