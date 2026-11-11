# Multi-Agent Business Process

## Why it matters

Agentic AI is moving from chat to business processes that agents carry out end to end: reading requests, looking up systems, drafting decisions and taking actions with human approval. Those processes need orchestration, tools, memory, approvals and tracing to be safe and debuggable. In this capstone you build an **orchestrated multi-agent process** that shows each of these. It's the flagship portfolio piece for engineers and a strong design case for architects.

**Brief:** Northwind's procurement team processes 400 purchase requests a week. Each request must be checked against policy and budget, priced from approved suppliers and approved by a manager above a threshold. Northwind wants agents to do the groundwork, with humans approving every commitment.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Intake agent extracts structured data from an email or form (item, quantity, cost centre, justification) |
| R2 | Policy agent checks the request against procurement policy (RAG over policy documents) |
| R3 | Supplier agent queries a supplier catalogue API and proposes the best quote (tool calls) |
| R4 | Budget check via a finance API (read-only) |
| R5 | Orchestrated workflow with clear handoffs; deterministic steps where possible |
| R6 | Human approval before any purchase order is created (write action); approval via Teams or email |
| R7 | Memory: the process can pause for days awaiting approval and resume with state |
| R8 | End-to-end tracing of every agent step, tool call and decision |
| R9 | Evaluation of task adherence, tool-call accuracy and intent resolution on 50 scenarios |
| R10 | Each agent has its own identity with least-privilege access to tools |

### Skills practised

Topics [18](../18-agents-with-foundry-agent-service/README.md), [19](../19-agent-frameworks-and-protocols/README.md), [11](../11-integration-messaging-and-apis/README.md), [22](../22-evaluation-observability-and-genaiops/README.md), [23](../23-responsible-ai-security-and-governance/README.md) and [34 file 04](../34-ai-solution-architecture/04-agentic-architecture.md).

## How it works on Azure

### Architecture

```text
Email / form ─▶ Intake agent ─▶ Workflow (Microsoft Agent Framework workflow, hosted on Foundry or Container Apps)
                                 ├─▶ Policy agent (file search / AI Search knowledge)
                                 ├─▶ Supplier agent (OpenAPI tool → API Management → catalogue API)
                                 ├─▶ Budget step (deterministic function → finance API, read-only)
                                 ├─▶ Summary + recommendation
                                 ├─▶ Human-in-the-loop approval (request info / approval event; Teams card or Logic Apps)
                                 └─▶ On approval: create PO via ERP API (write tool, separate identity)
State: checkpointed workflow state (Cosmos DB or framework checkpoint store)
Telemetry: OpenTelemetry → Application Insights / Foundry tracing
```

### Orchestration choice

| Pattern | Use here? |
|---|---|
| Sequential | Yes: intake → policy → supplier → budget → approval |
| Concurrent | Optional: run policy and supplier checks in parallel to cut latency |
| Handoff | For exceptions, such as handing off to a human buyer for unusual items |
| Group chat or magentic | Not needed; adds cost and unpredictability |

Use deterministic code for the budget check and purchase order creation. Use agents where language understanding or judgement is needed.

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Process map (as-is and to-be), agent responsibilities, tool inventory with read or write classification |
| M2 | Mock APIs (catalogue, finance, ERP) behind API Management |
| M3 | Single agents working individually with their tools |
| M4 | Workflow orchestration with checkpointing |
| M5 | Human approval and resume after a delay |
| M6 | Tracing, evaluation set and results |
| M7 | Security: per-agent identities, approval enforcement, prompt injection tests on email input |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Foundry Agent Service vs Agent Framework hosted yourself | Foundry for managed hosting, tools and tracing; self-hosted Agent Framework for custom runtime or portability |
| Agents vs deterministic steps | Keep money-moving actions deterministic and approved |
| One agent with many tools vs many agents | Split when tools, permissions or instructions differ significantly |
| Approval channel | Teams adaptive cards for user experience; Logic Apps for auditability and integration |

## Hands-on

1. Complete milestones M1–M7.
2. Show a trace where a request is rejected by policy, with the reason.
3. Prove that no purchase order can be created without approval, even with a prompt injection in the email.
4. Measure cost and latency per request; optimise one of them by 30%.
5. Write ADRs for orchestration pattern, hosting and approval design.

## Security, cost and reliability notes

- Separate identities for read tools and write tools; the write tool only accepts calls carrying a valid approval token.
- Validate tool inputs server-side (amount limits, allowed suppliers).
- Set timeouts, retries and maximum iterations per agent to avoid runaway loops and cost.
- Make approval idempotent so duplicate events don't create duplicate orders.

## Common mistakes

| Mistake | Fix |
|---|---|
| Letting an agent decide whether approval is needed | Enforce approval in the workflow or tool |
| Free-form multi-agent chat for a structured process | Sequential workflow with explicit handoffs |
| No checkpointing | Persist state for long waits |
| No evaluation | Scenario set with expected tool calls |

## Check yourself

1. Which steps should be deterministic, and why?
2. How do you guarantee approval before a write action?
3. Which orchestration pattern fits this process?
4. What does checkpointing enable?
5. Which evaluators matter for agents?

<details>
<summary>Answers</summary>

1. Budget checks and PO creation, because they move money and need predictable, auditable behaviour.
2. Enforce it outside the model: the workflow gates the step and the write API requires an approval token or record.
3. Sequential, optionally with concurrent checks and a handoff for exceptions.
4. Pausing for days awaiting approval and resuming with state after restarts.
5. Task adherence, tool-call accuracy, intent resolution, plus safety evaluators.

</details>

## References

- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/)
- [AI agent orchestration patterns](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Foundry Agent Service overview](https://learn.microsoft.com/azure/foundry/agents/overview)
- [get-started-with-ai-agents sample](https://github.com/Azure-Samples/get-started-with-ai-agents)
- [Agent evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/agent-evaluators)
