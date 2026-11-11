# Agent Framework Workflows

## Why it matters

Single agents are flexible but unpredictable; business processes need structure: this step, then that one; these three in parallel; stop for a human; resume after a crash. **Agent Framework workflows** give you explicit, inspectable execution paths that combine agents, ordinary functions, state, events and human input. They implement the standard multi-agent patterns (sequential, concurrent, handoff, group chat, Magentic), support checkpointing and human-in-the-loop, and can be defined in code or YAML. With Foundry's visual workflow designer retiring on 1 December 2026, this is Microsoft's recommended orchestration path.

## Key concepts

### Two APIs over one run model

| | **Functional** (`@workflow`, Python, experimental) | **Graph** (`WorkflowBuilder`) |
|---|---|---|
| Control flow | Native Python (`if`, loops, `asyncio.gather`) | Executors connected by edges and conditions |
| Best for | Sequential pipelines, custom loops, ad-hoc parallelism | Fixed graphs, fan-out/fan-in, type-validated routing |
| Parallelism | `asyncio.gather` | Parallel edge groups, superstep execution |
| Observability | Per-step events with `@step` | Per-executor events |
| Human-in-the-loop | `ctx.request_info()` | `RequestInfoExecutor` / request ports |
| Checkpointing | Per-step result caching | Superstep-boundary checkpoints |
| As an agent | `.as_agent()` | `.as_agent()` |

.NET and Go use the graph API.

### Graph primitives

| Primitive | Meaning |
|---|---|
| **Executor** | A processing unit (function, class or agent) that receives input and produces output |
| **Edge** | Connects one executor's output to another's input; can be conditional, fan-out or fan-in |
| **Event** | Workflow and executor events streamed during a run (progress, outputs, requests) |
| **State** | Shared workflow state managed by the runtime |
| **Superstep** | A round of executor execution; checkpoints are taken at superstep boundaries |
| **Sub-workflow** | A workflow used as an executor inside another |

### Built-in orchestrations

All live in `agent_framework.orchestrations` (Python) and `AgentWorkflowBuilder` (.NET):

| Builder | Pattern | Notes |
|---|---|---|
| `SequentialBuilder(participants=[...])` | Pipeline; each agent sees history and adds output | Supports tool approval; can mix custom executors |
| `ConcurrentBuilder(participants=[...])` | Fan-out to all, aggregate results | Independent perspectives |
| `HandoffBuilder(participants=[...]).with_start_agent(triage)` | Agents transfer control to specialists | Interactive back-and-forth with the user |
| `GroupChatBuilder(participants=[...])` | Manager selects next speaker in a shared conversation | Maker-checker, collaboration |
| Magentic | Group chat with a planning manager (from AutoGen's **Magentic-One**) that tracks progress and adapts | Complex, open-ended tasks |

### Human-in-the-loop

Executors call `ctx.request_info(request_data=..., response_type=...)`; the workflow emits a request event and pauses; your app sends a response, which is routed to a method decorated with `@response_handler`. Agent-level tool approvals also surface as requests.

### Checkpoints and resume

| Storage | Scope | Use |
|---|---|---|
| `InMemoryCheckpointStorage` | In-process | Tests, demos |
| `FileCheckpointStorage` | Local disk | Single machine, development |
| `CosmosCheckpointStorage` | Azure Cosmos DB | Production, distributed |

Pass `checkpoint_storage=` to `WorkflowBuilder`; checkpoints capture executor state and pending messages at superstep boundaries (plus entry checkpoints from Python 1.13), so you can resume after faults, deployments or long human waits.

### Declarative workflows

Define workflows in **YAML**, including agents, actions, conditions and variables. They're portable and versionable, and are the migration path for exported Foundry workflow YAML ([Topic 18 file 09](../18-agents-with-foundry-agent-service/09-multi-agent-workflows.md)).

### Workflows as agents

`workflow.as_agent()` wraps a workflow so it can be called like any agent: hosted in Foundry, exposed over Responses or A2A, or nested in another workflow.

## How it works on Azure

Handoff triage (Python):

```python
from agent_framework.foundry import FoundryChatClient
from agent_framework.orchestrations import HandoffBuilder
from azure.identity import AzureCliCredential

client = FoundryChatClient(project_endpoint=ENDPOINT, model="gpt-5-mini", credential=AzureCliCredential())
triage = client.as_agent(name="triage", instructions="Route to refunds, orders or returns. Don't answer yourself.")
refunds = client.as_agent(name="refunds", instructions="Handle refund requests under policy R-12.")
orders = client.as_agent(name="orders", instructions="Answer order status questions.")
returns = client.as_agent(name="returns", instructions="Arrange returns.")

workflow = (HandoffBuilder(participants=[triage, refunds, orders, returns])
            .with_start_agent(triage)
            .build())
```

Custom graph with checkpoints:

```python
from agent_framework import WorkflowBuilder, FileCheckpointStorage

storage = FileCheckpointStorage("./checkpoints")
builder = WorkflowBuilder(start_executor=extract, checkpoint_storage=storage)
builder.add_edge(extract, validate)
builder.add_edge(validate, approve)        # approve uses ctx.request_info for a human decision
builder.add_edge(approve, post_to_erp)
workflow = builder.build()
```

Deploy by wrapping the workflow as an agent and hosting it as a **Foundry hosted agent** (endpoint, identity, scaling, tracing).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Workflow vs single agent** | Workflows when steps, order, approvals or audit trail matter; single agent for open-ended conversational tasks. |
| **Functional vs graph** | Functional for straightforward Python pipelines; graph for fan-out/fan-in, typed routing, and .NET/Go. |
| **Built-in orchestration vs custom graph** | Start with built-in builders; drop to `WorkflowBuilder` when you need deterministic business steps mixed with agents. |
| **Checkpoint storage** | Cosmos DB for production and multi-instance hosting; file or memory for development. |
| **Code vs YAML** | Code for complex logic and tests; YAML for portability, reviews by non-developers and Foundry workflow migration. |
| **Group chat vs Magentic** | Group chat for a few collaborating agents with simple speaker selection; Magentic for complex tasks needing planning and progress tracking (higher cost). |
| **Determinism** | Put deterministic rules in executors (code), not in agent prompts. |

## Hands-on

1. Build a **sequential** writer → reviewer workflow and run it:

```python
from agent_framework.orchestrations import SequentialBuilder
workflow = SequentialBuilder(participants=[writer, reviewer]).build()
events = await workflow.run("Write a tagline for a budget-friendly eBike.")
for msg in events.get_outputs()[0].messages:
    print(msg.author_name, ":", msg.text)
```

2. Change it to **concurrent** with `ConcurrentBuilder(participants=[legal, marketing, finance])` and compare outputs and latency.
3. Build the **handoff** triage workflow above and chat through a refund scenario.
4. Add a **human approval** executor using `ctx.request_info` before any refund over £100; persist checkpoints with `FileCheckpointStorage`; stop the process mid-wait and resume from the latest checkpoint.
5. Visualise the workflow with the framework's visualisation support or DevUI, and inspect events in OpenTelemetry traces.

## Security, cost and reliability notes

- **Approvals** on consequential steps; record who approved what (events and traces).
- **State contains data**: checkpoints may hold personal or confidential data; secure Cosmos DB or disk storage accordingly.
- **Cost**: concurrent and group-chat patterns multiply model calls; Magentic adds planning overhead. Right-size models per participant.
- **Failure handling**: time-outs, retries and validation between steps; checkpoints for recovery.
- **Version upgrades**: checkpoint formats changed in Python 1.13; read upgrade guides before upgrading long-running workflows.

## Common mistakes

| Mistake | Fix |
|---|---|
| Encoding business rules in agent prompts | Use executors (code) and conditional edges |
| Group chat for a fixed pipeline | Sequential |
| No checkpointing for workflows that wait on humans | Add checkpoint storage |
| In-memory checkpoints in production | Cosmos DB storage |
| Rebuilding retired Foundry workflows from scratch | Reuse exported YAML with declarative workflows |
| Treating the workflow as untestable | Unit-test executors; evaluate agent steps with rubrics |

## Check yourself

1. Compare the functional and graph workflow APIs.
2. Define executor, edge and superstep.
3. Name the five built-in orchestration patterns and their builders.
4. How does a Python executor ask a human for input?
5. Which checkpoint storage options exist, and which suits production?
6. What are declarative workflows used for?
7. What does `.as_agent()` enable?
8. Where does Magentic orchestration come from, and when is it worth its cost?

<details>
<summary>Answers</summary>

1. Functional uses native Python control flow with `@workflow`/`@step` (experimental, Python only); graph uses `WorkflowBuilder` with executors and edges, supports fan-out/fan-in, typed routing and superstep checkpoints across languages.
2. Executor: a processing unit; edge: a connection routing output to input (optionally conditional); superstep: one round of execution after which checkpoints are taken.
3. Sequential (`SequentialBuilder`), concurrent (`ConcurrentBuilder`), handoff (`HandoffBuilder`), group chat (`GroupChatBuilder`), Magentic (Magentic builder/manager).
4. `ctx.request_info(...)`, with the response handled by a `@response_handler` method.
5. In-memory, file and Cosmos DB; Cosmos DB for production.
6. Defining workflows in YAML for readability, portability and versioning, including migrating exported Foundry workflows.
7. Wrapping a workflow so it can be called like an agent: hosted in Foundry, exposed over protocols, or nested.
8. From AutoGen's Magentic-One; for complex, open-ended tasks needing dynamic planning and delegation.

</details>

## References

- [Workflow concepts](https://learn.microsoft.com/agent-framework/concepts/workflows/)
- [Workflow builder and execution](https://learn.microsoft.com/agent-framework/concepts/workflows/builder-and-execution)
- [Orchestrations: sequential, concurrent, handoff, group chat, Magentic](https://learn.microsoft.com/agent-framework/workflows/orchestrations/sequential)
- [Human-in-the-loop](https://learn.microsoft.com/agent-framework/workflows/human-in-the-loop)
- [Checkpoints and resuming](https://learn.microsoft.com/agent-framework/workflows/checkpoints)
- [Declarative workflows](https://learn.microsoft.com/agent-framework/workflows/declarative)
