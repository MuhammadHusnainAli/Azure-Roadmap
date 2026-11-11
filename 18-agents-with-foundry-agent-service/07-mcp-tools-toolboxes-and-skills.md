# MCP Tools, Toolboxes and Skills

## Why it matters

The **Model Context Protocol (MCP)** has become the standard way to plug tools into agents: GitHub, Azure DevOps, Microsoft 365, databases and thousands of SaaS services publish MCP servers. But connecting every agent directly to every server duplicates credentials, scatters governance and bloats context windows. Foundry adds three layers on top: the **MCP tool** for direct connections with approvals, **Toolboxes** to curate tools once and expose them through one governed MCP endpoint, and **Skills** to package reusable know-how. **Tool search** keeps large tool sets usable. For platform teams this is the backbone of an internal "agent tool catalogue".

## Key concepts

### MCP tool (direct)

Connect a prompt agent to a remote MCP server with `MCPTool`:

| Setting | Purpose |
|---|---|
| `server_label` | Name for the server in traces and approvals |
| `server_url` | Remote MCP endpoint (public, or private via a dedicated MCP subnet) |
| `allowed_tools` | Restrict which server tools the agent may call. **An empty list allows all tools.** |
| `require_approval` | `always`, `never`, or per-tool rules; approval produces an `mcp_approval_request` item that your app answers with `mcp_approval_response` |
| `project_connection_id` | Connection holding credentials (key, Entra or OAuth) |
| custom headers | For servers needing extra headers; review what you send |

Authentication options for MCP and other tool connections:

| Option | Use |
|---|---|
| Key-based | Third-party servers with API keys or PATs |
| **Microsoft Entra** (agent identity or project managed identity) | Azure-hosted servers (Functions, Container Apps, APIM) |
| **OAuth identity passthrough (On-Behalf-Of)** | Act with the signed-in user's permissions |
| Unauthenticated | Only where appropriate (public, read-only) |

Behaviour notes: synchronous MCP calls time out after **100 seconds**; use **background mode** for longer operations (model permitting). Private MCP servers run on Azure Container Apps with internal ingress on a dedicated subnet delegated to `Microsoft.App/environments`.

### Toolbox

A **toolbox** is a managed, versioned collection of tools exposed through **one MCP-compatible endpoint**. Microsoft frames its value as four pillars:

| Pillar | Capability |
|---|---|
| **Build** | Curate tools and skills once; configure auth centrally |
| **Discover** | **Tool search** finds relevant tools at runtime from hundreds |
| **Consume** | Any MCP client (Foundry prompt or hosted agents, Agent Framework, LangGraph, your code) connects to one endpoint |
| **Govern** | Auth, authorisation, guardrails on tool inputs/outputs, observability, versioning |

Key properties:

- **Foundry-homed, not Foundry-bound**: usable by any MCP client.
- **Centralised authentication**: credential injection, token refresh and policy enforcement with Entra ID and OAuth passthrough; agents don't hold per-tool secrets.
- **Versioning**: create and test a new version at `/toolboxes/{name}/versions/{v}/mcp`, then **promote** it to default; agents on the consumer endpoint `/toolboxes/{name}/mcp` pick it up without code changes.
- **Supported tools**: MCP, web search, Azure AI Search, code interpreter, file search, OpenAPI, A2A, browser automation, Fabric IQ, Work IQ, skills, tool search, reminder. (Function calling, Bing grounding, computer use, image generation, SharePoint, Fabric data agent and Azure Functions are direct-only.)

### Tool search

With many tools, sending every definition on every call costs tokens, crowds the context and reduces selection accuracy. Tool search hides tools behind two meta-tools:

- `tool_search`: describe the need, get the most relevant tools.
- `call_tool`: invoke a discovered tool by name.

You can **pin** critical tools, add organisation-specific context to improve discovery, and auto-pin frequently used tools.

### Skills (Preview)

> Tools define *what* an agent can do; skills define *how* it performs a task.

A **skill** is a folder with a `SKILL.md` at its root (YAML front matter with `name` and `description`, then Markdown instructions), plus optional scripts, references and assets. Skills follow the open **Agent Skills** format, are stored centrally in Foundry, and are **versioned and immutable**.

| Use skills with | How |
|---|---|
| Prompt agents on the **Copilot harness** | Reference skills in the agent definition (up to 50) |
| **Toolboxes** | Attach a skill version; MCP clients discover it via MCP resources |
| **Hosted agents** | Download skill content and bundle it |
| Optimisation | Generate or improve skills from recorded conversations (skill optimisation) |

Skill names: lowercase letters, digits and hyphens, max 64 characters; description max 1,024 characters.

## How it works on Azure

Create a toolbox (Python):

```python
from azure.ai.projects.models import MCPToolboxTool, ToolSearchToolboxTool, WebSearchToolboxTool

tb = project.toolboxes.create_version(
    name="support-tools",
    description="Web search, GitHub issues and tool search for support agents",
    tools=[
        WebSearchToolboxTool(),
        MCPToolboxTool(server_label="github", server_url="https://api.githubcopilot.com/mcp/",
                       require_approval="never", project_connection_id="github-pat"),
        ToolSearchToolboxTool(),
    ])
# consumer endpoint (follows default version):
# {project_endpoint}/toolboxes/support-tools/mcp?api-version=v1
```

Consume it from an Agent Framework hosted agent with `FoundryToolbox` (Python) or `AddFoundryToolboxes` (.NET), which handles toolbox authentication and forwards the call context for user delegation; from a prompt agent, attach it as an MCP tool. Manage toolboxes with `azd ai toolbox list|show|versions list|delete`.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Direct MCP vs toolbox** | Direct for one agent and one server; toolbox when tools are shared, need central auth, governance or versioning, or exceed a handful. |
| **`allowed_tools`** | Always set a minimal list; an empty list exposes everything the server offers. |
| **Approvals** | `always` for write or sensitive actions and untrusted servers; `never` only for read-only, trusted tools. |
| **Auth** | Prefer Entra or OBO over shared keys; OBO when results must respect user permissions. |
| **Tool search** | Enable when a toolbox holds dozens of tools; pin the few critical ones. |
| **Skills vs instructions** | Put reusable procedures (escalation policy, review checklist, report format) in skills shared across agents; keep agent-specific behaviour in instructions. |
| **Third-party servers** | Treat as untrusted: vet the publisher, review data flows, monitor, and restrict tools. |

## Hands-on

1. **Direct MCP with approval.** Create a prompt agent with GitHub's MCP server and `require_approval="always"`:

```python
import os, json
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential(),
                          allow_preview=True)   # as in Microsoft's MCP sample
agent = project.agents.create_version(agent_name="gh-helper", definition=PromptAgentDefinition(
    model="gpt-5-mini", instructions="Use GitHub tools to answer questions about repositories.",
    tools=[MCPTool(server_label="github", server_url="https://api.githubcopilot.com/mcp",
                   allowed_tools=["get_me", "list_issues"], require_approval="always",
                   project_connection_id="github-pat")]))

with project.get_openai_client(agent_name=agent.name) as oai:
    conv = oai.conversations.create()
    r = oai.responses.create(conversation=conv.id, input="List open issues in contoso/web.")
    approvals = [{"type": "mcp_approval_response", "approval_request_id": i.id,
                  "approve": i.name in ("get_me", "list_issues")}
                 for i in r.output if i.type == "mcp_approval_request"]
    if approvals:
        r = oai.responses.create(conversation=conv.id, input=approvals)
    print(r.output_text)
```

2. **Toolbox.** Create the `support-tools` toolbox above, test its versioned endpoint, then connect two different agents to the consumer endpoint. Create version 2 with an extra tool, test it, promote it, and confirm both agents see it without redeployment.

3. **Skill.** Author `SKILL.md`:

```markdown
---
name: incident-summary
description: Summarise an incident into impact, timeline, root cause and actions.
---
# Incident summary
Produce four sections: Impact, Timeline (UTC), Root cause, Actions (owner, due date).
Never speculate about root cause; write "under investigation" if unknown.
```

Upload it with the Foundry Toolkit for VS Code, attach it to the toolbox, and invoke it from an agent.

## Security, cost and reliability notes

- **MCP risks**: Microsoft warns MCP implementations are susceptible to attacks, cascading failures and loss of human oversight. Vet servers, restrict tools, use approvals and monitor.
- **Data sharing**: prompts, arguments and headers go to the MCP server; third-party servers are governed by their own terms.
- **Tool poisoning**: tool descriptions from a server enter the model context; a malicious description can steer the agent. Review tool lists when servers change.
- **Guardrails** can be applied to tool inputs and outputs at the toolbox level.
- **Cost**: every tool definition adds input tokens; tool search reduces this for large toolboxes.
- **Reliability**: 100-second synchronous timeout; background mode for long operations; handle server outages gracefully.

## Common mistakes

| Mistake | Fix |
|---|---|
| `allowed_tools=[]` thinking it blocks all tools | Empty means all; list allowed tools explicitly |
| Each agent storing its own GitHub token | Use a toolbox with central auth |
| Approvals disabled for write tools | `require_approval="always"` for side effects |
| Hundreds of tools without tool search | Enable tool search and pin critical tools |
| Changing a toolbox in place for all agents | Test the new version on its versioned endpoint, then promote |
| Copying the same procedure into many agents' instructions | Package it as a skill |

## Check yourself

1. What does an empty `allowed_tools` list mean?
2. How does an app approve an MCP tool call?
3. Name the four toolbox pillars.
4. What's the difference between a toolbox's versioned and consumer endpoints?
5. What problem does tool search solve, and how?
6. What is a skill, and what file defines it?
7. Which authentication option makes MCP results respect the user's permissions?
8. What happens to MCP calls that exceed 100 seconds?

<details>
<summary>Answers</summary>

1. All tools exposed by the MCP server are allowed.
2. The response contains `mcp_approval_request` items; the app sends `mcp_approval_response` items with `approve` true or false in the same conversation.
3. Build, Discover, Consume, Govern.
4. The versioned endpoint (`/versions/{v}/mcp`) serves a specific version for testing; the consumer endpoint (`/mcp`) always serves the default version.
5. Large tool sets cost tokens, crowd context and reduce selection accuracy; tool search hides tools behind `tool_search` and `call_tool` meta-tools that discover and invoke relevant tools on demand.
6. A versioned, immutable package of instructions, scripts and resources defined by a `SKILL.md` file with name and description front matter.
7. OAuth identity passthrough (On-Behalf-Of).
8. They time out unless run in background mode with a supporting model.

</details>

## References

- [Connect to MCP servers](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
- [MCP authentication](https://learn.microsoft.com/azure/foundry/agents/how-to/mcp-authentication)
- [What is Toolbox in Foundry?](https://learn.microsoft.com/azure/foundry/agents/concepts/toolbox-overview)
- [Create and manage a toolbox](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/toolbox)
- [Tool search](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/tool-search)
- [Use skills in Foundry (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/skills)
- [Model Context Protocol](https://modelcontextprotocol.io/)
