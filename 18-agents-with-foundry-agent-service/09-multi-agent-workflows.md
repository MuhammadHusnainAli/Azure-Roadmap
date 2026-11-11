# Multi-Agent Workflows

## Why it matters

As agent use cases grow, one agent with every tool and every instruction becomes slow, expensive and unreliable. Splitting work across specialised agents (a triage agent, a billing agent, a technical agent, a reviewer) can improve quality, security isolation and team ownership. It also adds latency, cost, failure points and debugging complexity. The orchestration options on Azure also changed in 2026: the classic **Connected Agents** tool isn't in the new Foundry Agent Service, and Foundry's visual **workflows retire on 1 December 2026**. This file explains when to go multi-agent and how to orchestrate it on today's platform.

## Key concepts

### Single or multi-agent? (Cloud Adoption Framework)

Start with a **multi-agent** system when:

| Criterion | Why |
|---|---|
| **Security or compliance boundaries** | Different data classifications need isolated agents with least-privilege access |
| **Multiple teams** | Separate teams own separate domains and release cycles |
| **Planned growth** | The roadmap adds many features, sources or business units |

Otherwise **test a single agent first**: distinct roles (planner, reviewer) don't automatically justify multiple agents; single agents are faster to build and cheaper (no repeated context across agents).

### Orchestration patterns (Azure Architecture Center)

| Pattern | How | Use when | Avoid when |
|---|---|---|---|
| **Sequential** | Agents run in a fixed order, each refining the previous output | Clear pipeline stages (draft → review → format) | Stages need back-and-forth |
| **Concurrent** | Agents work in parallel on the same input; results aggregated | Independent perspectives (legal, security, finance reviews) | Agents depend on each other's output |
| **Group chat** | A manager coordinates a shared conversation among agents | Collaborative problem solving, **maker-checker** loops | Large groups (chaos, cost) |
| **Handoff** | An agent transfers control to a more suitable specialist | Triage and routing (support escalation) | Routing is deterministic (use code) |
| **Magentic** | A manager plans and dynamically assigns tasks with a task ledger | Open-ended, complex tasks | Simple, predictable tasks |
| **Human-in-the-loop** | Workflow pauses for human approval or input | High-impact actions, compliance | Fully automated low-risk tasks |

### Options on Azure (late 2026)

| Option | Status | Notes |
|---|---|---|
| **Microsoft Agent Framework workflows** | Recommended | Graph-based workflows with sequential, concurrent, handoff, group chat and Magentic patterns, checkpoints and human-in-the-loop; code or declarative YAML; deploy as a hosted agent ([Topic 19 file 02](../19-agent-frameworks-and-protocols/02-agent-framework-workflows.md)) |
| **A2A tool** | `a2a` (A2A 1.0) **GA**; `a2a_preview` (0.3) Preview | One Foundry agent calls another agent (Foundry or external) via the Agent-to-Agent protocol; the caller keeps control of the conversation |
| **Azure Logic Apps** | GA | Low-code visual orchestration that calls Foundry agents as steps and mixes deterministic business logic |
| **Hosted agent with your own orchestration** | GA | LangGraph, CrewAI, custom code |
| **Foundry workflows** (visual designer) | Preview, **retiring 1 Dec 2026** | Export YAML and migrate to Agent Framework, Logic Apps or A2A |
| **Connected Agents** (classic API) | Not in new Agent Service | Replace with A2A or workflows |

### Human-in-the-loop

| Mechanism | Where |
|---|---|
| MCP tool approvals (`mcp_approval_request` / `mcp_approval_response`) | Prompt agents (file 07) |
| Agent Framework workflow request/response and checkpoints | Agent Framework |
| `@multi_turn_task` durable tasks that suspend and resume on the same `task_id` | Hosted agents (wait minutes to days; survives restarts) |
| Logic Apps approval actions | Logic Apps |

### Multi-agent costs and risks

Every hand-off adds latency and tokens (context is re-sent), every agent needs its own prompt engineering, evaluation and monitoring, and credentials and attack surface multiply. Microsoft's guidance: manual chaining is brittle; use a workflow engine that provides coordination, state, branching and transparency.

## How it works on Azure

**Calling another agent with the A2A tool:**

```python
from azure.ai.projects.models import A2ATool, A2AProtocolVersion, PromptAgentDefinition

conn = project.connections.get("billing-agent-a2a")   # target: .../agents/billing/endpoint/protocols/a2a
triage = project.agents.create_version(agent_name="triage", definition=PromptAgentDefinition(
    model="gpt-5-mini",
    instructions="Answer general questions. For invoices, refunds or payments, delegate to the billing agent.",
    tools=[A2ATool(a2a_version=A2AProtocolVersion.V1_0, project_connection_id=conn.id)]))
```

For a Foundry agent as the target, the connection target is the target's A2A base path (`https://{account}.services.ai.azure.com/api/projects/{project}/agents/{agent}/endpoint/protocols/a2a`) with audience `https://ai.azure.com`. App-only calls use the calling agent's identity; delegated user context can be preserved where required (file 13).

**Agent Framework sequential workflow** (see Topic 19 for the full set of patterns):

```python
from agent_framework.orchestrations import SequentialBuilder

workflow = SequentialBuilder(participants=[researcher, writer, reviewer]).build()
events = await workflow.run("Write a one-page brief on our Q3 churn drivers.")
final = events.get_outputs()[0]          # AgentResponse from the last participant
```

Deploy the workflow inside a hosted agent (file 04) so it gets a Foundry endpoint, identity and tracing.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **One agent or many** | Single agent until security boundaries, team ownership or scale force a split. |
| **Pattern** | Deterministic order → sequential; independent views → concurrent; triage → handoff; collaboration and review → group chat or maker-checker; open-ended → Magentic. |
| **A2A vs workflow** | A2A for simple delegation between two agents; workflows when you need ordering, branching, state or approvals. |
| **Agent Framework vs Logic Apps** | Agent Framework for developers and complex AI orchestration; Logic Apps for business-process automation with 1,000+ connectors and a visual designer. |
| **Context passed between agents** | Pass only what the next agent needs; summarise long histories. |
| **Failure handling** | Time-outs per agent, retries, fallback to a human, and checkpointing for long workflows. |
| **Evaluation** | Evaluate each agent and the end-to-end workflow; trace across agents. |

## Hands-on

Build a triage → specialist handoff with A2A.

1. Create a **billing** prompt agent with a function or OpenAPI tool for invoices.
2. Enable its **A2A endpoint** (incoming A2A, Preview) so other agents can call it.
3. Create a project connection to the billing agent's A2A base path with audience `https://ai.azure.com`.
4. Create the **triage** agent with `A2ATool` (above).
5. Test: "Why was I charged twice in October?" Confirm in the trace that triage delegated to billing and composed the answer.
6. Add a maker-checker step: a **reviewer** agent that checks the billing answer against policy before it's returned (implement as an Agent Framework sequential workflow hosted as a hosted agent).
7. Compare latency and tokens for the single-agent version (one agent with the billing tool) versus the two-agent version.

## Security, cost and reliability notes

- **Least privilege per agent** is the main security benefit of multi-agent design; don't give every agent every tool.
- **Trust between agents**: treat another agent's output like any tool output (untrusted data); validate before acting.
- **Identity**: decide whether downstream agents act as themselves or on behalf of the user; avoid privilege escalation through delegation chains.
- **Cost multiplies**: each agent call re-sends context; measure tokens per completed task.
- **Latency**: sequential hand-offs add up; use concurrency where possible and set time-outs.
- **Migration deadline**: export Foundry workflow YAML before 1 December 2026.

## Common mistakes

| Mistake | Fix |
|---|---|
| Multi-agent by default | Test a single agent first |
| One agent per "role" with no real boundary | Use personas or skills within one agent |
| Manual chaining in app code | Use a workflow engine |
| Starting new work on Foundry visual workflows | Agent Framework, Logic Apps or A2A |
| Looking for Connected Agents in the new API | Use the A2A tool |
| No human approval on high-impact actions | Add HITL steps |

## Check yourself

1. What three criteria from the Cloud Adoption Framework justify starting with multi-agent?
2. Describe the sequential, concurrent, group chat, handoff and Magentic patterns.
3. What replaced Connected Agents in the new Foundry Agent Service?
4. What happens to Foundry workflows on 1 December 2026, and what are the three migration options?
5. Which A2A tool type is GA, and which protocol version does it use?
6. How do hosted agents implement long-running human-in-the-loop waits?
7. Give three costs or risks that multi-agent designs add.
8. When would you use Logic Apps rather than Agent Framework?

<details>
<summary>Answers</summary>

1. Crossing security or compliance boundaries, multiple teams owning domains, and planned growth.
2. Sequential: fixed pipeline; concurrent: parallel independent work aggregated; group chat: managed shared conversation (including maker-checker); handoff: transfer control to a specialist; Magentic: manager plans and assigns tasks dynamically.
3. The A2A tool (and workflows).
4. They retire; migrate to Microsoft Agent Framework (recommended), Azure Logic Apps, or direct A2A connections, after exporting the workflow YAML.
5. `a2a` (`A2ATool`), using A2A protocol 1.0.
6. With durable `@multi_turn_task` tasks that suspend after asking for a decision and resume on the same `task_id`, surviving restarts.
7. Any three: added latency per hand-off, more tokens, separate prompt engineering, evaluation and monitoring per agent, larger attack surface and credential management.
8. For business-process automation that needs a visual designer, many prebuilt connectors and deterministic steps alongside AI.

</details>

## References

- [Single-agent or multi-agent systems (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/single-agent-multiple-agents)
- [AI agent orchestration patterns (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Connect to an A2A agent endpoint](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent)
- [Enable incoming A2A (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint)
- [Build a workflow in Foundry (retiring) and migration guide](https://learn.microsoft.com/azure/foundry/agents/concepts/workflow)
- [Add a human-in-the-loop approval step](https://learn.microsoft.com/azure/foundry/agents/how-to/add-human-in-the-loop)
- [Microsoft Agent Framework workflows](https://learn.microsoft.com/agent-framework/workflows/)
- [Sequential orchestration in Agent Framework](https://learn.microsoft.com/agent-framework/workflows/orchestrations/sequential)
