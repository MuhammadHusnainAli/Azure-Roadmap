# Migrating AI Workloads to Foundry

## Why it matters

The AI platform landscape moved fast between 2023 and 2026, and many organisations now run AI on a mix of approaches that are being superseded: **hub-based** Azure AI Foundry projects, the **OpenAI Assistants API** (threads and runs), **classic Foundry agents**, **Semantic Kernel** and archived **AutoGen** code, **prompt flow** (retiring 20 April 2027), standalone Azure OpenAI resources, retired model versions, and AI built on other clouds. Consolidating onto current **Microsoft Foundry** (Foundry resources and projects, the new Agent Service with the Responses API, Agent Framework, Foundry Models, guardrails, evaluation and the Control Plane) reduces risk from retirements and unlocks governance and new capabilities. This file maps each starting point to a migration path.

## Key concepts

### Starting points and paths

| From | To | Notes |
|---|---|---|
| **Hub-based projects** (Azure AI Foundry hubs on Azure ML) | **Foundry resource + Foundry projects** | Recreate connections, move assets, update SDK endpoints; hub features differ ([Topic 14 file 02](../14-microsoft-foundry-platform/02-foundry-resources-and-projects.md)) |
| **Standalone Azure OpenAI resources** | Foundry resource (Azure OpenAI capabilities are part of Foundry) | Upgrade/attach rather than rebuild where supported; keep deployments; adopt Foundry projects for agents and evaluation |
| **OpenAI Assistants API** | **Foundry Agent Service** (new agents) with **conversations** and **responses** | Migration tool available for Assistants → Agents; replace threads with conversations and runs with responses |
| **Classic Foundry agents** | New agents | Check tool availability, migrate definitions, replace threads/runs |
| **Agent Applications (legacy publishing)** | New agent object model | Agents now have a stable endpoint and unique Entra agent identity by default; publishing means distribution to Microsoft 365/Teams |
| **Semantic Kernel / AutoGen** | **Microsoft Agent Framework** | Official migration guides; AutoGen is archived ([Topic 19 file 03](../19-agent-frameworks-and-protocols/03-migrating-from-semantic-kernel-and-autogen.md)) |
| **Prompt flow** (Foundry and Azure ML) | Agent Framework (code) or Foundry agents/workflows, plus Foundry evaluation | Retires **20 April 2027**; no new development |
| **Retiring model versions** | Newer models via upgrade policies | Evaluate replacements before retirement ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)) |
| **Other clouds** (Amazon Bedrock, Google Vertex AI, OpenAI direct) | Foundry Models (OpenAI, Anthropic, Meta, Mistral, DeepSeek, xAI and more), Agent Service, Azure AI Search/Foundry IQ | Re-evaluate quality and cost; port prompts and tools; MCP/A2A help portability |
| **Self-built RAG stacks** | Azure AI Search / Foundry IQ knowledge bases | Re-index with integrated vectorisation; evaluate retrieval quality |

### Assistants/classic agents → new agents: minimum steps (Microsoft)

1. Compare current tools with new agent tool availability; choose replacements for tools that don't carry over.
2. Migrate the agent definition (classic agents or Assistants path).
3. Replace **threads with conversations** and **runs with responses**.
4. Verify: state, tool calls, outputs and error handling.

Use `azure-ai-projects` 2.3.0 or later for the new agents SDK; a migration tool can automate Assistants API migrations, after which you review generated assets.

### Migration principles

| Principle | Why |
|---|---|
| **Evaluate before and after** | Build an evaluation dataset from current production behaviour; compare old vs new ([Topic 22](../22-evaluation-observability-and-genaiops/README.md)) |
| **Agents as code** | Recreate definitions from Git, not by hand (Topic 27 file 06) |
| **Identity and governance** | Adopt Entra Agent ID, guardrails, Control Plane and Agent 365 as part of the move (Topic 23) |
| **Parallel run** | Shadow or canary traffic to the new implementation |
| **Data and state** | Decide whether conversation history and files migrate or start fresh |

## How it works on Azure

Example: Assistants API code → new agents with the Responses API (Python sketch):

```python
# Before (Assistants API): client.beta.assistants.create(...); client.beta.threads.create(); client.beta.threads.runs.create(...)
# After (Foundry Agent Service, new agents):
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
agent = project.agents.create_version(
    agent_name="support-agent",
    definition=PromptAgentDefinition(model="gpt-5-mini", instructions=open("instructions.md").read()),
)
with project.get_openai_client(agent_name=agent.name) as client:
    conversation = client.conversations.create()                  # replaces threads
    response = client.responses.create(conversation=conversation.id, input="Where is order 1042?")   # replaces runs
    print(response.output_text)
```

Migration plan for an AI estate:

```text
1. Inventory: AI resources (Resource Graph), SDK versions, APIs used (Assistants, prompt flow), models and versions, frameworks.
2. Prioritise by retirement dates: prompt flow (Apr 2027), retiring models, archived AutoGen, legacy agent applications.
3. Landing zone: Foundry resource/projects per environment, networking, guardrails, Application Insights, AI gateway.
4. Migrate per application: evaluation baseline → port code/definitions → evaluate → canary → cut over → retire old resources.
5. Govern: register agents (Control Plane, Agent 365), apply policies, monitoring and cost controls.
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prompt agents vs hosted agents vs Agent Framework code** | Prompt agents for configuration-driven agents; hosted agents or Agent Framework for code-heavy orchestration ([Topic 18](../18-agents-with-foundry-agent-service/README.md)). |
| **Prompt flow replacement** | Agent Framework workflows for code-first teams; Foundry agents/workflows where low-code fits; check Foundry workflow retirement plans before adopting. |
| **Cross-cloud model choice** | Use Foundry Models to keep a similar model family or switch based on evaluation and cost. |
| **History migration** | Usually start fresh; migrate only if regulatory or UX requires it. |
| **Timing** | Migrate before retirement deadlines with buffer for evaluation. |

## Hands-on

1. Inventory AI resources and SDK usage across subscriptions.
2. Migrate a sample Assistants API app to new agents (conversations and responses); compare outputs with an evaluation set.
3. Port a Semantic Kernel or AutoGen sample to Agent Framework using the official guide.
4. Rebuild a prompt flow as Agent Framework code and evaluate parity.
5. Write a migration plan ordered by retirement dates.

## Security, cost and reliability notes

- **Identity changes**: new agents get unique Entra agent identities; reassign permissions with least privilege.
- **Guardrails** must be configured explicitly for new agents (agent guardrails override model guardrails).
- **Cost**: run old and new in parallel briefly; model changes affect token costs.
- **Reliability**: keep the old path available until canary results are acceptable.

## Common mistakes

| Mistake | Fix |
|---|---|
| Migrating without an evaluation baseline | Evaluate before and after |
| Rebuilding agents by hand in the portal | Definitions as code |
| Waiting until retirement dates | Plan early |
| Ignoring tool availability differences | Compare tools first |
| Moving to another deprecated pattern | Check current lifecycle status |

## Check yourself

1. Map five legacy AI starting points to their Foundry targets.
2. What are Microsoft's four minimum steps for migrating Assistants or classic agents?
3. When does prompt flow retire, and what replaces it?
4. What changed with the new agent object model?
5. Why build an evaluation baseline first?
6. How should you prioritise an AI migration backlog?

<details>
<summary>Answers</summary>

1. Hubs → Foundry resource/projects; Assistants API → new agents with conversations/responses; Semantic Kernel/AutoGen → Agent Framework; prompt flow → Agent Framework or Foundry agents; other clouds → Foundry Models and Agent Service.
2. Compare tools and choose replacements; migrate the agent definition; replace threads with conversations and runs with responses; verify state, tool calls, outputs and errors.
3. 20 April 2027; Microsoft Agent Framework (or Foundry agents/workflows) with Foundry evaluation.
4. Agents themselves have stable endpoints and unique Entra identities; Agent Applications and Deployments are gone; publishing means distributing to Microsoft 365/Teams.
5. To prove the new implementation matches or improves quality and safety.
6. By retirement dates, business criticality and risk, then by effort.

</details>

## References

- [Migrate to the new agents developer experience](https://learn.microsoft.com/azure/foundry/agents/how-to/migrate)
- [Migrate from agent applications](https://learn.microsoft.com/azure/foundry/agents/how-to/migrate-agent-applications)
- [Migrate hub-based projects](https://learn.microsoft.com/azure/foundry-classic/how-to/migrate-project)
- [Agent Framework migration from Semantic Kernel](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/)
- [Agent Framework migration from AutoGen](https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/)
- [Model migration](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/model-migration)
