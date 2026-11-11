# Agentic Architecture

## Why it matters

"Should this be one agent, five agents, or no agent at all?" is now one of the most consequential AI design questions. Agents that reason and call tools can automate whole processes, but each additional agent adds coordination overhead, latency, cost, new failure modes and new security boundaries. Microsoft's Architecture Center guidance is clear: **use the lowest level of complexity that reliably meets the requirement**, which is often a direct model call or a single agent with tools. This file covers the complexity spectrum, the main orchestration patterns (sequential, concurrent, group chat, handoff, magentic), workflow agents, and how to choose and implement a topology on Azure with Foundry Agent Service and Microsoft Agent Framework.

## Key concepts

### Complexity spectrum

| Level | Description | When to use | Considerations |
|---|---|---|---|
| **Direct model call** | One model call with a good prompt; no tools | Classification, summarisation, translation, extraction in one pass | Simplest; if prompting solves it, you don't need an agent |
| **Single agent with tools** | One agent reasoning over tools, knowledge and APIs, looping as needed | Varied requests in one domain needing dynamic tool use | Often the right enterprise default; easier to debug; set iteration limits |
| **Multi-agent orchestration** | Specialised agents coordinated by an orchestrator or protocol | Cross-domain problems, distinct security boundaries, parallel specialisation | Overhead, latency, failure modes; justify by prompt complexity, tool overload or security needs |

### Orchestration patterns

| Pattern | How it works | Use | Avoid when |
|---|---|---|---|
| **Sequential** | Agents in a fixed pipeline; each refines the previous output | Multistage processes with clear dependencies (draft → review → format) | Steps are independent or need dynamic routing |
| **Concurrent** | Agents work in parallel on the same input; results aggregated | Multiple perspectives, independent analyses, latency reduction | Agents need each other's outputs; aggregation is unclear |
| **Group chat** | Agents converse in a shared thread managed by a chat manager | Collaborative problem solving, **maker–checker** validation loops | Many agents (noisy), strict latency or cost budgets |
| **Handoff** | Agents transfer control to the most suitable specialist dynamically | Triage and routing (customer service), unclear upfront which expert is needed | Routing is static or loops risk ping-pong |
| **Magentic** | A manager agent builds and adapts a task ledger/plan and delegates to specialists | Open-ended complex tasks without a predetermined plan | Predictable, well-defined processes (use workflows) |

### Workflow vs agentic control

| Approach | Control | Fit |
|---|---|---|
| **Deterministic workflows** (Agent Framework workflows, Durable Functions, Logic Apps) | Code defines steps; agents perform bounded tasks | Business processes needing predictability, audit and approvals |
| **Agentic orchestration** | Model decides steps | Open-ended tasks |
| **Hybrid** | Workflow skeleton with agentic steps and human checkpoints | Most enterprise processes |

### Building blocks on Azure

| Need | Azure |
|---|---|
| Agent runtime | Foundry Agent Service (prompt agents, hosted agents) ([Topic 18](../18-agents-with-foundry-agent-service/README.md)) |
| Orchestration in code | Microsoft Agent Framework (agents, orchestrations, workflows) ([Topic 19](../19-agent-frameworks-and-protocols/README.md)) |
| Tools | Built-in tools, OpenAPI tools, Azure Functions, MCP servers, toolboxes |
| Inter-agent protocols | A2A for cross-platform agents; connected agents within Foundry |
| State and memory | Conversations, agent memory stores, Cosmos DB |
| Identity | Entra Agent ID per agent; on-behalf-of for user-scoped actions |
| Governance | Guardrails at tool call/response, approvals, Control Plane, Agent 365 (Topic 23) |
| Observability | OpenTelemetry multi-agent spans (`invoke_agent`, `execute_tool`) (Topic 22) |

## How it works on Azure

Example: insurance claim processing as a hybrid workflow:

```text
Workflow (Agent Framework workflow or Durable Functions):
 1. Intake agent (single agent + document extraction tool) → structured claim JSON
 2. Concurrent: policy-check agent | fraud-signal agent | coverage-calculation (deterministic code)
 3. Aggregator: rules decide auto-approve threshold
 4. Maker–checker: drafting agent writes decision letter; compliance agent reviews (group chat, max 2 rounds)
 5. Human approval step for payouts > £5,000 (approval UI)
 6. Payment API call (tool with approval token), notification
Tracing across all steps; evaluation per agent; guardrails on tool calls; agent identities with least privilege
```

Sequential orchestration with Agent Framework (Python sketch):

```python
from agent_framework.orchestrations import SequentialBuilder

workflow = SequentialBuilder(participants=[extractor_agent, policy_agent, letter_agent]).build()
result = await workflow.run("Process claim CLM-2026-0042 using the attached documents")
```

Choosing a topology: start with a single agent; split into multiple agents only when the evaluation shows prompt overload, tool confusion (too many tools), or when separate security boundaries (different identities/data access) are required.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Complexity level** | Lowest level that meets quality; prove need for multi-agent with evaluation. |
| **Determinism** | Use workflows for steps that must happen in order or be audited. |
| **Context sharing** | Pass only necessary context between agents; large shared context raises cost and leakage risk. |
| **Failure handling** | Timeouts, iteration limits, fallbacks, human escalation; idempotent tools. |
| **Security boundaries** | Separate agents (and identities) for untrusted-content reading vs privileged actions. |
| **Cost and latency** | Each agent hop adds tokens and time; concurrent patterns reduce latency at higher cost. |

## Hands-on

1. Take a business process and classify each step as direct call, single agent, deterministic code or human.
2. Build a single agent with tools; measure task completion; identify whether multi-agent is needed.
3. Implement a sequential and a concurrent orchestration with Agent Framework; compare latency and cost.
4. Add a maker–checker loop with a maximum number of rounds.
5. Trace a multi-agent run and evaluate each agent separately.

## Security, cost and reliability notes

- **Cascading failures** (OWASP agentic ASI08): one agent's error propagates; validate inter-agent outputs.
- **Least agency**: each agent gets only its tools and data.
- **Cost controls**: max steps, token budgets per run, alerts on anomalies.
- **Reliability**: workflows need checkpointing and resumability for long-running processes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Multi-agent by default | Start with one agent |
| Fully autonomous financial actions | Workflows with approvals |
| Unbounded loops between agents | Iteration limits |
| One identity for all agents | Per-agent identities |
| Evaluating only the final answer | Per-agent and per-step evaluation |

## Check yourself

1. Describe the three complexity levels.
2. Explain sequential, concurrent, group chat, handoff and magentic patterns.
3. When are deterministic workflows better than agentic orchestration?
4. What justifies moving from one agent to many?
5. Which Azure services implement agent runtime and orchestration?
6. Name three reliability and cost controls for agentic systems.

<details>
<summary>Answers</summary>

1. Direct model call (single prompt), single agent with tools (default for many enterprise cases), multi-agent orchestration (specialised agents with coordination).
2. Sequential: fixed pipeline; concurrent: parallel agents with aggregation; group chat: shared conversation managed by a chat manager (maker–checker); handoff: dynamic transfer to specialists; magentic: manager agent plans and delegates adaptively.
3. When steps must be predictable, ordered, auditable or include approvals.
4. Evidence of prompt overload, tool confusion, or need for distinct security boundaries or parallel specialisation.
5. Foundry Agent Service and Microsoft Agent Framework (plus tools, A2A, MCP).
6. Any three: iteration limits, timeouts, token budgets, fallbacks, human escalation, checkpointing, idempotent tools, anomaly alerts.

</details>

## References

- [AI agent orchestration patterns](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Microsoft Agent Framework](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [Foundry Agent Service overview](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Application design for AI workloads](https://learn.microsoft.com/azure/well-architected/ai/application-design)
