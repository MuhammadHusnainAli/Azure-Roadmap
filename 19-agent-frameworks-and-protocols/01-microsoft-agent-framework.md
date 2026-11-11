# Microsoft Agent Framework

## Why it matters

Microsoft Agent Framework is Microsoft's open-source SDK for building agents and multi-agent workflows in code. It is the successor to both **Semantic Kernel** (enterprise-grade agent and plugin foundations) and **AutoGen** (multi-agent research patterns), unified into one framework. It powers hosted agents in Foundry, is the recommended replacement for Foundry's retiring visual workflows, and works with many model providers, not just Azure. If you write agent code on the Microsoft stack, this is the default library; architects need to know where it fits next to Foundry Agent Service, Copilot Studio and third-party frameworks.

## Key concepts

### Four areas

| Area | What it provides |
|---|---|
| **Agents** | Individual agents that use LLMs, call tools and MCP servers, and generate responses; providers include Microsoft Foundry, Azure OpenAI, OpenAI, Anthropic, Ollama, Amazon Bedrock, Google Gemini, Mistral and more |
| **Harness agent** | An opinionated, "batteries-included" agent for long, multi-step tasks: planning and to-dos, context compaction, file access and memory, standing tool approvals, observability |
| **Workflows** | Functional and graph-based workflows connecting agents and functions through explicit execution paths (file 02) |
| **Integrations** | Model providers, agent services (Foundry, Copilot Studio, GitHub Copilot, A2A), tools (Foundry Toolbox, shell), context providers, middleware (Purview), evaluation, UI (AG-UI, ChatKit, DevUI) |

Languages: **Python** and **.NET** (C#); **Go** in public preview. Python packages: `agent-framework` (metapackage) or `agent-framework-foundry` (core + Foundry integration). .NET: `Microsoft.Agents.AI.*` packages.

### Building blocks

| Block | Role |
|---|---|
| **Chat client** | Connects to a model (`FoundryChatClient`, Azure OpenAI, OpenAI, Anthropic clients) |
| **Agent** | `Agent(client=..., instructions=..., tools=[...])` in Python; `AIAgent` in .NET |
| **Session** | `agent.create_session()` holds conversation state across `run` calls |
| **Tools** | Python functions with `@tool` (with `approval_mode`), hosted tools (code interpreter, file search, web search), local and hosted **MCP** tools |
| **Context providers** | Run around each invocation to add context (memory, RAG) and process results; built-in integrations for Azure AI Search, Cosmos DB, Redis, Mem0, Neo4j, Foundry memory and more |
| **Middleware** | Agent-run, function-calling and chat-client middleware for logging, validation, guardrails, termination, result overrides |
| **Compaction** | Strategies to keep history within budgets (experimental; [Topic 16 file 03](../16-prompt-engineering-and-model-customisation/03-context-engineering.md)) |
| **Observability** | OpenTelemetry traces, metrics and logs by default in the harness |
| **Declarative agents** | YAML/JSON agent definitions (`kind: Prompt`) loaded with `AgentFactory` |
| **Skills** | Agent Skills packages loaded at runtime |

### Harness agent capabilities

| Capability | Default |
|---|---|
| Function invocation with iteration limit | On |
| Per-service-call history persistence | On |
| Compaction | When token limits or a strategy are supplied |
| To-do tracking, plan and execute modes | On |
| Session file memory | On (shared file access opt-in) |
| Tool approval with standing approvals and auto-approval rules | On |
| OpenTelemetry | On |

### Where it runs

| Hosting option | Notes |
|---|---|
| **Foundry hosted agents** | `ResponsesHostServer` exposes an agent over the Responses protocol; Foundry provides endpoint, identity, scaling ([Topic 18 file 04](../18-agents-with-foundry-agent-service/04-hosted-agents.md)) |
| **Durable extension** | Azure Functions Durable for long-running, reliable agents |
| **Self-hosting** | OpenAI Responses or OpenAI-compatible endpoints, A2A server, MCP server, Telegram and others |
| **Local / any container** | Your own App Service, Container Apps or AKS |

### Relationship to Foundry Agent Service

| Use | When |
|---|---|
| **Foundry prompt agent** | Configuration-only agent run by Foundry |
| **Agent Framework + Foundry models** | Code-first agent calling Foundry models and tools via `FoundryChatClient` and the Foundry Toolbox |
| **Agent Framework as a Foundry hosted agent** | Code-first agent with Foundry-managed hosting, identity and publishing |
| **Agent Framework calling a Foundry agent** | Use a Foundry prompt agent as a component inside a larger code workflow |

## How it works on Azure

Minimal agent with a tool and a session (Python):

```python
import asyncio
from agent_framework import Agent, tool
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential

@tool(approval_mode="never_require")          # read-only tool; use "always_require" for side effects
def get_order_status(order_id: str) -> str:
    """Get the delivery status of a Contoso order."""
    return f"Order {order_id}: out for delivery, ETA today 17:00."

async def main():
    agent = Agent(
        client=FoundryChatClient(
            project_endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
            model="gpt-5-mini",
            credential=AzureCliCredential()),
        name="order-helper",
        instructions="Help customers with order status. Always look up the order.",
        tools=[get_order_status])
    session = agent.create_session()
    print(await agent.run("Where is order 4471?", session=session))
    print(await agent.run("And when will it arrive?", session=session))

asyncio.run(main())
```

Function middleware for logging or guardrails:

```python
from agent_framework import FunctionMiddleware, FunctionInvocationContext

class AuditFunctions(FunctionMiddleware):
    async def process(self, context: FunctionInvocationContext, call_next):
        print("calling", context.function.name, context.arguments)
        await call_next()
        print("result", context.result)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Agent Framework vs Foundry prompt agents** | Prompt agents for configuration-only agents; Agent Framework when you need code: custom tools in-process, middleware, workflows, non-Foundry models or custom hosting. |
| **Plain agent vs harness agent** | Plain agent for chat and tool use; harness for long, multi-step work needing planning, files, compaction and standing approvals. |
| **Python vs .NET** | Follow your team's stack; feature parity is close, with occasional language-specific gaps (check each feature's tabs). Go is preview. |
| **Provider choice** | `FoundryChatClient` for Foundry models and tools; other providers are first-class too, which helps multi-cloud and local (Ollama, Foundry Local) scenarios. |
| **Memory** | Context providers backed by Foundry memory, Cosmos DB, Redis or Azure AI Search; keep state outside the process for hosted agents. |
| **Tool approvals** | Use `approval_mode="always_require"` for side-effecting tools in production. |
| **Hosting** | Foundry hosted agents for managed endpoint and identity; Durable Functions for long-running reliability; your own containers for full control. |

## Hands-on

1. `pip install agent-framework-foundry azure-identity`; `az login`.
2. Run the order-helper sample; then add a second tool `issue_refund(order_id, amount)` with `approval_mode="always_require"` and handle the approval request in your loop.
3. Add the `AuditFunctions` middleware and observe calls.
4. Replace `FoundryChatClient` with another provider (for example a local Ollama model) without changing the agent code; note what changes.
5. Wrap the agent with `ResponsesHostServer` and deploy it as a Foundry hosted agent with `azd` ([Topic 18 file 04](../18-agents-with-foundry-agent-service/04-hosted-agents.md)).
6. Enable OpenTelemetry export to Application Insights and inspect the trace of one run.

## Security, cost and reliability notes

- **Credentials**: `DefaultAzureCredential` is convenient for development; in production, Microsoft recommends a specific credential such as `ManagedIdentityCredential`.
- **Environment files**: Agent Framework doesn't auto-load `.env`; keep secrets out of code and use Key Vault or managed identity.
- **Approvals and guardrails**: use tool approval and middleware-based termination for risky actions; Purview middleware can apply data protection policies.
- **Iteration limits**: cap function-invocation loops to control cost.
- **Versioning**: the framework moves quickly; pin package versions and read upgrade guides (for example 2026 Python changes, MCP 2.x migration).
- **Observability**: keep OpenTelemetry on in production for debugging and evaluation.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting new projects on Semantic Kernel or AutoGen | Use Agent Framework (file 03) |
| Using code where a prompt agent suffices | Prefer prompt agents for configuration-only needs |
| Side-effect tools with `never_require` approval | Require approval |
| In-memory state in a hosted or scaled-out agent | Use sessions with persistent storage or context providers |
| Unpinned packages | Pin versions; follow upgrade guides |
| `DefaultAzureCredential` everywhere in production | Use managed identity credentials |

## Check yourself

1. What are the four main areas of Agent Framework?
2. Which earlier Microsoft frameworks does it succeed?
3. What does the harness agent add over a plain agent?
4. What are the three kinds of middleware?
5. What are context providers used for?
6. Name three ways to host an Agent Framework agent.
7. When would you use Agent Framework rather than a Foundry prompt agent?
8. Which languages are supported and at what status?

<details>
<summary>Answers</summary>

1. Agents, the harness agent, workflows and integrations.
2. Semantic Kernel and AutoGen.
3. Planning and to-dos, plan/execute modes, compaction, session file memory, standing tool approvals and default observability for long multi-step tasks.
4. Agent-run middleware, function-calling middleware and chat-client middleware.
5. Adding context before each invocation and processing results after (memory, RAG, enrichment).
6. Any three: Foundry hosted agents, Azure Functions durable extension, self-hosted Responses/OpenAI-compatible endpoints, A2A server, MCP server, your own containers.
7. When you need code: in-process tools, middleware, workflows, non-Foundry models or custom hosting.
8. Python and .NET (C#); Go in public preview.

</details>

## References

- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/)
- [Your first agent](https://learn.microsoft.com/agent-framework/get-started/your-first-agent)
- [Agent harness](https://learn.microsoft.com/agent-framework/concepts/harness)
- [Agent middleware](https://learn.microsoft.com/agent-framework/concepts/agents/middleware/)
- [Context providers](https://learn.microsoft.com/agent-framework/concepts/agents/conversations/context-providers)
- [Foundry hosted agents with Agent Framework](https://learn.microsoft.com/agent-framework/hosting/foundry-hosted-agent)
- [Agent Framework on GitHub](https://github.com/microsoft/agent-framework)
