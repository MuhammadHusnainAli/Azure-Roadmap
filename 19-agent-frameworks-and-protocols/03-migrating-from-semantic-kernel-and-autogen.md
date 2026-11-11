# Migrating from Semantic Kernel and AutoGen

## Why it matters

Many organisations built their first agents and copilots on **Semantic Kernel** (SK) or **AutoGen**. Both are now superseded by **Microsoft Agent Framework**, built by the same teams. AutoGen's repository is **archived**: Microsoft no longer provides bug fixes, security fixes, features or support, and advises against production use. Semantic Kernel v1.x remains supported for critical bugs and security issues, but new capabilities land in Agent Framework (which Microsoft describes as effectively "Semantic Kernel v2.0"). Engineers need a migration plan; architects need to judge urgency and risk; consultants will be asked "do we have to rewrite?".

## Key concepts

### Status (late 2026)

| Framework | Status | Guidance |
|---|---|---|
| **Microsoft Agent Framework** | **1.0 GA** (Python, .NET); Go preview | Default for new projects |
| **Semantic Kernel v1.x** | Supported for critical bugs and security; feature development has moved to Agent Framework | Existing projects can continue; plan migration, especially for agent features |
| **AutoGen** | **Archived**; no Microsoft fixes or support | Migrate; avoid in production |

### What changes from Semantic Kernel

| Area | Semantic Kernel | Agent Framework |
|---|---|---|
| Namespaces (.NET) | `Microsoft.SemanticKernel`, `Microsoft.SemanticKernel.Agents` | `Microsoft.Agents.AI` (+ `Microsoft.Extensions.AI` types) |
| Packages (Python) | `semantic_kernel` | `agent_framework`, `agent_framework.foundry`, provider packages |
| Agent creation | Every agent depends on a `Kernel`; Foundry agents created in the cloud first, then wrapped | One-line creation via provider extensions (`chatClient.AsAIAgent(...)`, `Agent(client=...)`) |
| Agent types | Many (`ChatCompletionAgent`, `AzureAIAgent`, `OpenAIAssistantAgent`, …) | Mostly one `Agent` over any chat client; `BaseAgent` for custom agents |
| Threads | Caller creates provider-specific thread types | Agent creates the session: `agent.create_session()` |
| Tools | `[KernelFunction]`/`@kernel_function` + plugin + kernel | Register directly: `tools=[fn]`, `@tool`, `AIFunctionFactory.Create` |
| Invocation | `InvokeAsync` returning streams of items | `RunAsync`/`run` with `AgentRunOptions`; `RunStreamingAsync`/`run_stream` |
| Options and DI | Kernel-based configuration | Options objects; standard dependency injection |
| Orchestration | SK agent group chat, process framework | Workflows and orchestration builders (file 02) |

**Compatibility bridge (Python)**: existing SK `KernelFunction`s can be converted with `.as_agent_framework_tool()`, and SK `VectorStore` search functions can be reused, so plugins don't need rewriting on day one.

### What changes from AutoGen

| Area | AutoGen | Agent Framework |
|---|---|---|
| Agent | `AssistantAgent(name, model_client, tools)`; single-turn unless `max_tool_iterations` increased | `Agent(name, client, tools)`; **multi-turn by default**, keeps calling tools until done |
| Tools | `FunctionTool` wrappers | `@tool` with schema inference, plus hosted tools (code interpreter, web search) |
| MCP | AutoGen MCP extensions | Local and hosted MCP tools |
| Agent as tool | `AgentTool` | `agent.as_tool()` |
| Middleware | — | Agent, function and chat middleware |
| Teams | `RoundRobinGroupChat`, `SelectorGroupChat`, `Swarm`, `MagenticOneGroupChat` | Group chat, handoff and Magentic orchestrations |
| GraphFlow | Control-flow graph; messages broadcast; edges are transitions | **Data-flow** `Workflow`: typed messages along edges; executors activate when inputs are ready |
| Runtime | Embedded and experimental distributed runtimes | Single-process composition today; distributed execution planned; host in Foundry or Durable Functions |
| Responses API | — | Supported |

### Migration strategies

| Strategy | When |
|---|---|
| **Leave and contain** | Stable SK app with no new agent features; keep on SK v1.x, patch, revisit annually |
| **Bridge** | Reuse SK plugins/functions inside new Agent Framework agents via the compatibility layer |
| **Incremental (strangler)** | Migrate one agent or workflow at a time behind a stable API |
| **Rewrite** | AutoGen-based production systems (archived dependency), or when moving to Foundry hosted agents |

## How it works on Azure

Migration often coincides with platform moves:

| From | To |
|---|---|
| SK `AzureAIAgent` on classic Azure AI Agent Service (threads/runs) | Agent Framework with `FoundryChatClient` or Foundry prompt agents on the new agents API (conversations/responses) |
| OpenAI Assistants API (threads) | Responses API conversations (the Assistants model is deprecated) |
| Self-hosted AutoGen teams | Agent Framework workflows hosted as Foundry hosted agents or Durable Functions |
| Foundry visual workflows (retiring 1 Dec 2026) | Agent Framework declarative workflows from exported YAML |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Urgency** | AutoGen in production: high (no security fixes). SK: medium; driven by need for new features, Foundry integration, or the Responses API. |
| **Big bang vs incremental** | Incremental behind stable interfaces; migrate tools first, then agents, then orchestration. |
| **Tests first** | Capture current behaviour with a scenario evaluation set before migrating; compare after. |
| **Platform alignment** | Combine framework migration with moving to Foundry hosted agents, Entra agent identity and toolboxes where it simplifies operations. |
| **Language** | Keep the team's language; Python and .NET are both GA. |
| **Feature gaps** | AutoGen's distributed runtime has no direct equivalent yet; use hosting options (Foundry, Durable Functions) or keep distribution in your application layer. |

## Hands-on

Migrate a small SK Python agent.

**Before (Semantic Kernel):**

```python
from semantic_kernel.agents import ChatCompletionAgent
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.functions import kernel_function

class Orders:
    @kernel_function(name="get_order_status", description="Get order status by ID")
    def get_order_status(self, order_id: str) -> str:
        return f"Order {order_id}: shipped"

agent = ChatCompletionAgent(service=AzureChatCompletion(), name="helper",
                            instructions="Help with orders.", plugins=[Orders()])
```

**After (Agent Framework):**

```python
from agent_framework import Agent, tool
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential

@tool(approval_mode="never_require")
def get_order_status(order_id: str) -> str:
    """Get order status by ID."""
    return f"Order {order_id}: shipped"

agent = Agent(client=FoundryChatClient(project_endpoint=ENDPOINT, model="gpt-5-mini",
                                       credential=AzureCliCredential()),
              name="helper", instructions="Help with orders.", tools=[get_order_status])
session = agent.create_session()
print(await agent.run("Where is order 4471?", session=session))
```

Then:

1. Run the same 20 test questions through both versions; compare answers with an evaluator.
2. Convert an existing `@kernel_function` with `.as_agent_framework_tool()` instead of rewriting it.
3. For an AutoGen `RoundRobinGroupChat`, re-implement it with `GroupChatBuilder` (file 02) and compare transcripts.

## Security, cost and reliability notes

- **Archived dependencies are a security risk**: AutoGen receives no security fixes; prioritise migrating production workloads and scanning dependencies.
- **Behaviour changes**: Agent Framework agents are multi-turn by default and keep calling tools; add iteration limits and approvals so migrated agents don't take more actions than before.
- **Credentials**: use managed identity credentials in production rather than `DefaultAzureCredential` everywhere.
- **Cost**: re-validate token usage after migration; different defaults (multi-turn tool loops, Responses API) change consumption.
- **Observability**: turn on OpenTelemetry in the new stack before cut-over to compare.

## Common mistakes

| Mistake | Fix |
|---|---|
| Keeping AutoGen in production "because it works" | Migrate; it's archived |
| Rewriting all SK plugins up front | Bridge with `.as_agent_framework_tool()` |
| Migrating without a behavioural baseline | Build a scenario evaluation set first |
| Assuming identical tool-loop behaviour | Set iteration limits and approvals |
| Migrating framework but staying on deprecated Assistants threads | Move to Responses conversations at the same time |

## Check yourself

1. What is the support status of Semantic Kernel and AutoGen?
2. Name four API differences between SK agents and Agent Framework agents.
3. How can you reuse SK plugins during migration?
4. How does Agent Framework's default agent behaviour differ from AutoGen's `AssistantAgent`?
5. Contrast AutoGen GraphFlow with Agent Framework workflows.
6. Which AutoGen team types map to which Agent Framework orchestrations?
7. Why should migration start with an evaluation baseline?
8. Which AutoGen capability has no direct equivalent yet?

<details>
<summary>Answers</summary>

1. SK v1.x is supported for critical bugs and security with features moving to Agent Framework; AutoGen is archived with no Microsoft fixes or support.
2. Any four: namespaces/packages; no Kernel dependency for agents; one `Agent` type instead of many; agent-created sessions instead of caller-created threads; direct tool registration instead of plugins; `run`/`RunAsync` instead of `InvokeAsync`.
3. Convert `KernelFunction`s with `.as_agent_framework_tool()` (and reuse VectorStore search functions).
4. Agent Framework agents are multi-turn by default and keep invoking tools until they can answer; AutoGen's `AssistantAgent` is single-turn unless configured.
5. GraphFlow is control-flow based, broadcasting messages with edges as transitions; Agent Framework workflows are data-flow based, routing typed messages along edges and activating executors when inputs are ready.
6. RoundRobin/Selector group chats → group chat orchestration; Swarm → handoff; MagenticOneGroupChat → Magentic orchestration.
7. To detect behavioural regressions (answers, tool calls, cost) between old and new implementations.
8. AutoGen's distributed runtime; Agent Framework focuses on single-process composition with distributed execution planned.

</details>

## References

- [Semantic Kernel to Agent Framework migration guide](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/)
- [AutoGen to Agent Framework migration guide](https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/)
- [Semantic Kernel and Microsoft Agent Framework (support FAQ)](https://devblogs.microsoft.com/agent-framework/semantic-kernel-and-microsoft-agent-framework/)
- [AutoGen repository (archived)](https://github.com/microsoft/autogen)
- [Microsoft Agent Framework repository](https://github.com/microsoft/agent-framework)
