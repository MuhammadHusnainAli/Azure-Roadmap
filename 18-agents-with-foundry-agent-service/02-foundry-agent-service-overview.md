# Foundry Agent Service Overview

## Why it matters

Foundry Agent Service (formerly Azure AI Agent Service) is Microsoft's managed platform for building, running and governing agents on Azure. It decides a lot for you: where conversation state lives, how tools authenticate, how agents scale, how they are versioned and published. Picking the right agent type and environment setup at the start determines your data residency, network isolation, compliance story and how much code you maintain. Architects must get this right; consultants must be able to explain it in one slide.

## Key concepts

### Platform at a glance

| Component | What it does |
|---|---|
| **Agent runtime** | Hosts and scales prompt agents and hosted agents; manages conversations, tool calls and lifecycle |
| **Toolboxes** | Curate tools once (web search, file search, code interpreter, MCP servers, functions) and share them through one managed MCP endpoint with central auth, governance and versioning |
| **Models** | Agent-compatible models from the Foundry catalogue; swap models without changing agent code |
| **Observability** | Tracing, metrics, evaluations, Application Insights integration |
| **Optimisation** | Agent optimiser (Preview) proposes better instructions, skills, tool descriptions and model choices |
| **Identity and security** | Microsoft Entra identity per agent, RBAC, content filters, virtual network isolation |
| **Publishing** | Versioning, stable endpoints, distribution to Microsoft Teams, Microsoft 365 Copilot and the Entra Agent Registry |

Foundry Agent Service is **GA**; some sub-features are in preview.

### Agent types

| Type | What you provide | What Foundry runs | Best for | File |
|---|---|---|---|---|
| **Prompt agent** | Instructions, model, tools (portal, SDK, REST or azd) | Everything; no code or compute to manage | Fast start; production agents without custom orchestration | 03 |
| **Voice-based prompt agent** (Preview) | Model, instructions, audio settings, tools | Voice orchestration via Voice Live; real-time WebSocket | Call centres, spoken assistants | 11 |
| **Hosted agent** | Your code (Agent Framework, LangGraph, OpenAI Agents SDK, Anthropic Agent SDK, custom) as a container or .zip | Container hosting, endpoint, scaling, identity | Custom code, custom orchestration, multi-agent systems | 04 |

Plus two related options:

- **Ephemeral agents**: call the **Responses API** directly with instructions and tools defined in your app code; no persisted agent resource, but you still get catalogue models, platform tools, OBO auth and project observability.
- **Microsoft Copilot harness** (Preview): an optional managed runtime for a *new* prompt agent that adds multi-step file, shell, web, subagent and skill workflows, operated by Foundry.

**Workflows (Preview) are retiring on 1 December 2026.** Foundry's visual workflow builder (sequential, group chat, human-in-the-loop) is being retired; Microsoft directs new multi-agent orchestration to **Microsoft Agent Framework** workflows (file 09, [Topic 19 file 02](../19-agent-frameworks-and-protocols/02-agent-framework-workflows.md)).

### Runtime components

| Component | Role | Use when |
|---|---|---|
| **Agent** | Persisted, versioned definition: model, instructions, tools, parameters | Many requests need the same behaviour |
| **Conversation** | Durable object storing **items** (messages, tool calls, tool outputs) | Later turns need server-side history |
| **Response** | One execution of a model or agent against input, producing output items | Every interaction |

You can chain turns with a conversation, with `previous_response_id`, or by resending items. If a conversation exceeds the model's context window, the input is truncated for that call; the conversation itself isn't. This replaces the older Assistants-style threads and runs used in the classic agent service.

### Environment setups

| | **Basic** | **Standard (public networking)** | **Standard with private networking (BYO VNet)** |
|---|---|---|---|
| Agent state storage | Microsoft-managed, logically separated | **Your** Cosmos DB, Storage, AI Search | Your resources |
| Customer-managed keys | No | Yes | Yes |
| Outbound network isolation | No | No | **Yes**: delegated subnet, no public egress |
| Effort | Minutes | 30–45 minutes to provision | Most |

Standard setup BYO resources:

| Resource | Stores |
|---|---|
| Azure Cosmos DB for NoSQL (≥ 3,000 RU/s total) | Conversations, agent definitions and run state (`agent-definitions-v1`, `run-state-v1` in the new runtime) |
| Azure Storage | Uploaded files |
| Azure AI Search | Vector stores created by agents |

Inbound private access (private endpoint, public access disabled on the Foundry account) works with any setup; **outbound** isolation requires standard setup with private networking. Projects are the unit of isolation: agents in one project share file storage, conversation storage and indexes.

### Roles

| Action | Role |
|---|---|
| Create the Foundry account and project | Foundry Account Owner |
| Standard setup: assign roles on Cosmos DB, Search, Storage | Role Based Access Control Administrator (or Owner) |
| Create and edit agents | Foundry User |
| Call agent endpoints only | **Foundry Agent Consumer** |

### Key limits (defaults)

| Limit | Value |
|---|---|
| Tools registered per agent | 128 |
| Versions per agent | 1,000 |
| Files per agent/conversation | 10,000 |
| File size for prompt agents | 512 MB |
| Total uploaded files | 300 GB |
| Messages per thread | 100,000 |
| Voice session duration | 60 minutes |

Model rate limits apply separately at the deployment level; 429s are common under load ([Topic 15 file 13](../15-foundry-models-and-azure-openai/13-rate-limits-quotas-and-resilience.md)).

## How it works on Azure

```text
Foundry resource (Microsoft.CognitiveServices/accounts)
 └─ Project ─────────────── agents (versions) · conversations · files · vector stores · connections · toolboxes
      │                     identity: project MI; each agent gets an Entra agent identity
      ├─ model deployments (Foundry Models)
      ├─ capability settings → BYO Cosmos DB / Storage / AI Search (standard setup)
      └─ delegated subnet (private networking)
```

Developers use the `azure-ai-projects` SDK (2.x) to manage agents and `project.get_openai_client()` to run responses and conversations through the OpenAI-compatible API. The development lifecycle is: create → test in the playground → trace → evaluate → optimise → publish → monitor.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prompt vs hosted** | Prompt agents unless you need custom code inside the agent loop, a specific framework, or custom protocols. |
| **Persisted agent vs ephemeral Responses calls** | Persisted agents for governance, versioning, publishing and optimisation; ephemeral when the agent definition should ship with app code. |
| **Basic vs standard** | Basic for prototypes; standard for production with data residency, CMK and audit requirements. |
| **Private networking** | Required for regulated workloads reaching private resources; plan subnet sizing and DNS. Web grounding tools still use public endpoints. |
| **Classic vs new** | New builds should use the new agents API (agents, conversations, responses); classic threads/runs and classic containers differ. |
| **Workflows** | Don't start new projects on the retiring Foundry workflows designer; use Agent Framework. |

## Hands-on

1. In the Foundry portal (new), create a project and deploy `gpt-5-mini`.
2. **Build → Agents → Create agent**; set instructions and add the web search tool; test in the playground.
3. Recreate the same agent in code and invoke it:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, WebSearchTool

project = AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
                          credential=DefaultAzureCredential())
agent = project.agents.create_version(
    agent_name="researcher",
    definition=PromptAgentDefinition(model="gpt-5-mini",
                                     instructions="Research questions on the web and cite sources.",
                                     tools=[WebSearchTool()]))
oai = project.get_openai_client(agent_name=agent.name)
conv = oai.conversations.create()
r1 = oai.responses.create(conversation=conv.id, input="What is Foundry IQ?")
r2 = oai.responses.create(conversation=conv.id, input="How is it billed?")
print(r2.output_text)
for item in oai.conversations.items.list(conversation_id=conv.id):
    print(item.type)
```

4. Compare with a standard setup: review the Bicep templates in the Foundry samples repository for standard and network-secured agent setups, and list the resources they create.

## Security, cost and reliability notes

- **Data location** depends on setup: basic uses Microsoft-managed storage; standard keeps conversations, files and vectors in your subscription.
- **Least privilege**: give app users Foundry Agent Consumer rather than Foundry User where they only need to call agents.
- **Content safety** filters help with prompt injection, including cross-prompt injection; keep them on.
- **Costs**: model tokens, tool usage (search, web grounding, code interpreter), hosted-agent compute, BYO resources (Cosmos DB RU/s, Search tier, Storage). Agent instances themselves aren't charged.
- **Reliability**: handle `session_quota_exceeded` and regional quota 429s with backoff; consider provisioned throughput for steady, high-volume agents.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building production agents on basic setup when data must stay in your tenant | Use standard setup |
| Starting new work on Foundry workflows | Use Agent Framework workflows |
| Hosted agents for simple configuration-only agents | Use prompt agents |
| Giving all app users Foundry User | Use Foundry Agent Consumer for callers |
| Assuming private endpoints cover every tool | Web grounding and some external tools still use public endpoints |
| Mixing classic and new API samples | Use the new agents, conversations and responses model |

## Check yourself

1. Name the three agent types and what Foundry manages for each.
2. What is an ephemeral agent?
3. What happens to Foundry workflows on 1 December 2026, and what replaces them?
4. Describe agents, conversations and responses.
5. Compare basic, standard and standard-with-private-networking setups.
6. Which three BYO resources does standard setup use, and what does each store?
7. Which role should an application user who only calls agents have?
8. Give three default Agent Service limits.

<details>
<summary>Answers</summary>

1. Prompt agents (Foundry runs everything), voice-based prompt agents (Foundry and Voice Live run voice orchestration), hosted agents (Foundry hosts your container: endpoint, scaling, identity).
2. An agent defined in your app code and executed via direct Responses API calls, with no persisted agent resource in Foundry.
3. They retire; Microsoft directs new orchestration to Microsoft Agent Framework workflows and provides migration paths.
4. Agent: persisted, versioned definition; conversation: durable store of items across turns; response: one execution producing output items.
5. Basic: Microsoft-managed storage, quick; standard: your own Cosmos DB, Storage and AI Search, CMK; standard with private networking: also outbound isolation via a delegated subnet.
6. Cosmos DB (conversations, agent definitions, run state), Storage (files), AI Search (vector stores).
7. Foundry Agent Consumer.
8. Any three: 128 tools per agent, 1,000 versions per agent, 10,000 files, 512 MB file size, 300 GB total files, 100,000 messages per thread, 60-minute voice sessions.

</details>

## References

- [What is Foundry Agent Service?](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Set up your environment](https://learn.microsoft.com/azure/foundry/agents/environment-setup)
- [Standard agent setup](https://learn.microsoft.com/azure/foundry/agents/concepts/standard-agent-setup)
- [Agents, conversations and responses](https://learn.microsoft.com/azure/foundry/agents/concepts/runtime-components)
- [Private networking for Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [Limits, quotas and regions](https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions)
- [Build a workflow in Foundry (retiring)](https://learn.microsoft.com/azure/foundry/agents/concepts/workflow)
