# Prompt Agents

## Why it matters

Most production agents don't need custom orchestration code. A model, clear instructions and the right tools, run by a managed runtime, cover support assistants, research helpers, internal Q&A with actions, and many line-of-business agents. **Prompt agents** are Foundry's declarative agent type: you configure them, Foundry runs them, and you get versioning, a stable endpoint, an Entra identity, tracing and evaluation without operating any compute. They are the default starting point Microsoft recommends, and the fastest way to deliver value.

## Key concepts

### What a prompt agent is

A **saved, versioned resource** in a Foundry project whose definition contains:

| Part | Purpose |
|---|---|
| **Model** | Reasoning and language; support varies by region and tool |
| **Instructions** | Role, behaviour, boundaries, expected output ([Topic 16 file 01](../16-prompt-engineering-and-model-customisation/01-prompt-design-fundamentals.md)) |
| **Tools** | Web search, file search, code interpreter, functions, OpenAPI, MCP, knowledge (files 05–07) |
| **Skills** | Reusable packages of instructions, scripts and resources for specialised tasks (file 07) |
| **Toolboxes** | Reusable tool sets behind a managed MCP endpoint |
| **Harness choice** | Default runtime or the Copilot harness (Preview), chosen at creation |
| **Other settings** | Model parameters supported by the model (for example reasoning effort), structured output format |

### How invocation works

1. Client sends input to the agent's Responses endpoint (by agent name).
2. Foundry loads the saved definition for the selected version and routes to the runtime.
3. The runtime coordinates model and tool calls, possibly many, until done.
4. Foundry returns the response, records traces, and appends items to the conversation if one is used.

### Versioning

| Property | Behaviour |
|---|---|
| **Immutable versions** | Every save creates a new version; saved versions can't change |
| **Drafts** | Test unsaved changes in the playground; they're lost if you leave |
| **Reference** | Code refers to agents as `<agent_name>:<version>`; the name can't be changed after creation |
| **Comparison** | Compare versions by setup, chat output on the same input, and YAML definition |
| **Active version** | The agent's stable endpoint serves an active version: "always use latest" or a pinned version via `version_selector` |
| **Limit** | Up to 1,000 versions per agent |

Microsoft's guidance for code: **create once and reuse** (get-or-create on start-up), treat a 404 on a known agent as "deleted, recreate", and don't recreate agents on every run.

### Agent endpoint and identity (new object model)

Agents created under the current object model automatically get:

- A **stable endpoint**: `https://{account}.services.ai.azure.com/api/projects/{project}/agents/{agent}/endpoint/protocols/openai/responses` (plus an Activity Protocol endpoint).
- A unique **Microsoft Entra agent identity** and **agent blueprint** (file 13).
- An **agent card** describing capabilities (used for A2A, file 12).
- A **version selector** controlling which version the endpoint serves.

"Publishing" now means distributing the agent to Microsoft 365 Copilot and Teams (file 12). The older **Agent Application** resource model is legacy; existing applications keep working, but new agents don't need it.

### Default runtime vs Copilot harness (Preview)

| | Default prompt-agent runtime | **Copilot harness** (Preview) |
|---|---|---|
| Use when | Instructions + model + declared tools are enough | Multi-step tool, skill, file or command workflows |
| Built-ins | Declared tools only | File read/write, shell, web fetch, subagents, plus declared tools and skills |
| Workspace | None | Conversation-scoped sandbox managed by Foundry |
| Constraints | Standard | Chosen at creation only; conversation required on every request; no `strict` function schemas; no BYO model; some parameters (temperature, tool_choice, max_output_tokens…) are harness-managed; limited regions |

### Authoring paths

| Path | Best for |
|---|---|
| **Foundry portal** | Interactive build, playground testing, quick evaluations |
| **SDK / REST** (`azure-ai-projects`, `PromptAgentDefinition`) | Source-controlled definitions in CI/CD |
| **Azure Developer CLI** (`azd` with Foundry agent extensions) | Agent definition in source control, deploy and invoke from the CLI |

## How it works on Azure

Typical code shape:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, FileSearchTool

project = AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential())
agent = project.agents.create_version(
    agent_name="hr-helper",
    definition=PromptAgentDefinition(
        model="gpt-5-mini",
        instructions=INSTRUCTIONS,
        tools=[FileSearchTool(vector_store_ids=[vector_store_id])]))
oai = project.get_openai_client(agent_name=agent.name)   # responses go to this agent
```

The lifecycle (Microsoft's checklist): choose the agent type → create and test → add tools and data → save versions → debug with tracing → evaluate quality and safety → publish and integrate → monitor and iterate.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prompt agent vs hosted agent** | Prompt agent unless you need your own framework, middleware, custom protocol or code inside the loop (file 04). |
| **Prompt agent vs ephemeral Responses calls** | Saved agents give versioning, endpoints, identity, publishing and the optimiser; ephemeral calls keep the definition in app code. |
| **Default runtime vs Copilot harness** | Default for most agents; harness when the agent must work with files, run commands or delegate to subagents, and you accept preview constraints. |
| **"Always latest" vs pinned version** | Pin in production; promote by changing the version selector after evaluation passes. |
| **One agent vs several** | Keep each agent focused; split when instructions or tools sprawl (file 14). |
| **Instructions in the portal vs source** | Treat the definition as code; author in Git and deploy via SDK/azd ([Topic 16 file 04](../16-prompt-engineering-and-model-customisation/04-prompt-management.md)). |

## Hands-on

Create, version, compare and pin a prompt agent.

```python
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())

v1 = project.agents.create_version(agent_name="policy-helper", definition=PromptAgentDefinition(
    model="gpt-5-mini",
    instructions="Answer questions about Contoso's travel policy. Be brief."))

v2 = project.agents.create_version(agent_name="policy-helper", definition=PromptAgentDefinition(
    model="gpt-5-mini",
    instructions=("Answer questions about Contoso's travel policy in at most three bullet points. "
                  "If the question isn't about travel policy, say you can only help with travel policy.")))
print(v1.version, v2.version)

oai = project.get_openai_client(agent_name="policy-helper")
for q in ["Can I fly business class to New York?", "What's the capital of France?"]:
    print(q, "->", oai.responses.create(input=q).output_text)
```

Then:

1. In the portal, open the agent, compare v1 and v2 (setup, YAML, chat output on the same input).
2. Set **Active version** to v2 explicitly instead of "Always use latest".
3. Run a small evaluation (task adherence, intent resolution) against both versions and record the difference.
4. Call the agent's stable endpoint URL from a second app identity that has only the **Foundry Agent Consumer** role.

## Security, cost and reliability notes

- **Identity**: new agents get their own Entra agent identity; grant it only the roles its tools need. Legacy agents created before the object-model change share the project identity; recreate them to get a unique identity.
- **Tool approvals**: require approval for tools with side effects (MCP tools support `require_approval`).
- **Content filters** apply to agent model calls; configure guardrails at the deployment or agent level.
- **Cost**: per-call inference plus tool usage; no compute charge for prompt agents. Reasoning effort and number of tool calls drive cost.
- **Change control**: pin the active version in production; roll back by switching the version selector.

## Common mistakes

| Mistake | Fix |
|---|---|
| Recreating the agent on every app start | Get-or-create once; reuse the name/version |
| Production endpoint on "always use latest" | Pin and promote deliberately |
| Instructions edited only in the portal | Keep definitions in source control |
| Choosing hosted agents for simple configurations | Start with prompt agents |
| Enabling the Copilot harness for a simple Q&A agent | Use the default runtime |
| Forgetting that the agent identity needs roles for tools | Assign roles to the agent's identity, not just your user |

## Check yourself

1. What makes up a prompt agent definition?
2. What happens when you save a change to a prompt agent?
3. How does code refer to a specific agent version?
4. What does the version selector control?
5. What does a new agent receive automatically under the current object model?
6. When would you choose the Copilot harness, and name two of its preview constraints.
7. What is Microsoft's guidance on creating agents in application code?
8. Which role lets an app call an agent without being able to edit it?

<details>
<summary>Answers</summary>

1. Model, instructions, tools, skills, toolboxes, harness choice and supported settings.
2. A new immutable version is created; earlier versions are preserved in history.
3. `<agent_name>:<version>`.
4. Which agent version (or traffic split) the agent's stable endpoint serves.
5. A stable endpoint, a unique Entra agent identity and blueprint, an agent card and a version selector.
6. For multi-step file, shell, web or subagent workflows run by Foundry; constraints include creation-time only selection, conversation required on every request, no strict function schemas, no BYO model, limited regions.
7. Create once and reuse (get-or-create); treat 404 on a known agent as deleted and recreate; don't recreate on every run.
8. Foundry Agent Consumer.

</details>

## References

- [What are prompt agents?](https://learn.microsoft.com/azure/foundry/agents/concepts/prompt-agents)
- [Quickstart: create a prompt agent](https://learn.microsoft.com/azure/foundry/agents/quickstarts/prompt-agent)
- [Agent development lifecycle](https://learn.microsoft.com/azure/foundry/agents/concepts/development-lifecycle)
- [Migrate to the new agent endpoint and publishing experience](https://learn.microsoft.com/azure/foundry/agents/how-to/migrate-agent-applications)
- [Use the Copilot harness with prompt agents (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/use-tools-skills-copilot-harness)
- [Agent optimiser overview (Preview)](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-optimizer-overview)
