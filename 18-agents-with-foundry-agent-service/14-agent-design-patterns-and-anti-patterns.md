# Agent Design Patterns and Anti-Patterns

## Why it matters

Most agent failures aren't model failures; they're design failures: too many agents, too many tools, vague tool descriptions, no limits on loops, no plan for errors, no human checkpoint, no cost ceiling. This file pulls together Microsoft's guidance from the Azure Architecture Center, Cloud Adoption Framework and Foundry tool best practices into a practical checklist of patterns to use and anti-patterns to avoid. It is the file to revisit before every design review.

## Key concepts

### Start at the lowest level of complexity

| Level | Description | Use when |
|---|---|---|
| **Direct model call** | One well-crafted prompt, no tools | Classification, summarisation, translation, extraction in one pass |
| **Single agent with tools** | One agent loops over model and tool calls | **Often the right default for enterprise**: varied queries within one domain needing dynamic tool use. Set iteration limits. |
| **Multi-agent orchestration** | Specialised agents coordinated by an orchestrator or protocol | Cross-domain problems, distinct security boundaries, parallel specialisation, or when a single agent fails due to prompt complexity or tool overload |

### Patterns that work

| Pattern | Description |
|---|---|
| **Single agent, focused tools** | One domain, 5–15 well-described tools; toolbox with tool search if more |
| **Router / triage + specialists** | Deterministic or model-based routing to specialised agents (handoff) |
| **Maker-checker** | One agent produces, another verifies against criteria before release |
| **Plan-then-execute** | Reasoning model plans steps; cheaper model or tools execute; plan reviewed for risky tasks |
| **Retrieval-first** | Always ground on knowledge (Foundry IQ, search) before answering domain questions |
| **Human-in-the-loop on consequential actions** | Approvals for payments, deletions, external messages |
| **Deterministic shell, agentic core** | Workflow code for known steps; agent only where judgement is needed |
| **Model tiering** | Small models for classification, extraction and formatting; large or reasoning models only where needed |
| **Durable state** | Persist progress and checkpoints outside the context window for long tasks |

### Tool design (Foundry tool best practices)

| Practice | Why |
|---|---|
| Describe what each tool is for **and when to use it** in instructions | The model chooses tools from descriptions |
| Add decision rules for overlapping tools ("Use File Search before Web Search for internal content") | Prevents wrong tool choice |
| Use `tool_choice` (`auto`, `required`, `none`) for deterministic control | Forces or prevents tool calls when it matters |
| Task-shaped tools (`get_order_status`) over generic ones (`run_sql`) | Safer and easier to choose |
| Return only needed fields | Reduces tokens and injection surface |
| Tell the agent what to do when a tool fails or returns nothing | Graceful recovery |
| Treat tool outputs as untrusted; validate structured outputs; approve consequential actions | Security |
| For voice: concise tools, fast responses, interim messages | Latency |

### Reliability patterns (Architecture Center)

- Time-outs and retries per agent and tool.
- **Graceful degradation** when an agent or tool fails.
- **Surface errors** rather than hiding them.
- **Validate output** before passing it to the next agent; retry, ask for clarification or halt.
- **Circuit breakers** for agent dependencies.
- Isolation: avoid shared single points of failure (one model endpoint or knowledge store rate-limiting parallel agents).
- **Checkpoints** to resume after faults or deployments.

### Cost control

- Right-size the model per agent; monitor tokens per agent and per run.
- Compact context between agents and turns ([Topic 16 file 03](../16-prompt-engineering-and-model-customisation/03-context-engineering.md)).
- Limit iterations and tool calls; cap reasoning effort.
- Cache stable prefixes (prompt caching) and repeated retrievals where permissions allow.

### Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Agent for a deterministic process** | Slower, costlier, less reliable | Code or workflow |
| **Agent for static FAQ** | Unnecessary complexity | Classic RAG |
| **Multi-agent by default** | Coordination overhead, latency, failure modes | Single agent first |
| **Agents without meaningful specialisation** | Cost without benefit | Merge them |
| **Tool soup** (dozens of overlapping tools) | Wrong tool choice, token bloat | Focused tools, tool search |
| **God prompt** (pages of instructions) | Contradictions, drift | Skills, specialised agents, structure |
| **Unbounded loops** | Runaway cost | Iteration and time limits |
| **Shared mutable state between concurrent agents** | Inconsistent data | Explicit state ownership |
| **Deterministic pattern for non-deterministic work (and vice versa)** | Brittle or unpredictable | Match pattern to problem |
| **Ignoring latency of multi-hop chains** | Poor UX | Parallelise, fewer hops |
| **Growing context forever** | Cost and quality decay | Compaction and summaries |
| **Shared admin identity** | No least privilege or audit | Per-agent identity, OBO |
| **No evaluation** | Silent regressions | Scenario evaluations in CI and production |
| **Trusting tool or agent outputs blindly** | Injection and cascading errors | Validate and approve |

## How it works on Azure

| Concern | Foundry / Azure capability |
|---|---|
| Single agent with tools | Prompt agents (file 03), toolboxes and tool search (file 07) |
| Orchestration | Agent Framework workflows, A2A, Logic Apps (file 09) |
| Approvals | MCP approvals, durable tasks, workflow HITL |
| Limits | `max_tool_calls`, `max_output_tokens`, reasoning effort, agent instructions |
| Identity | Entra Agent ID, OBO (file 13) |
| Observability | Foundry tracing (OpenTelemetry) in Application Insights; agent monitoring dashboards; evaluations ([Topic 22](../Roadmap.md)) |
| Safety | Content filters, Prompt Shields and XPIA protection; guardrails on toolbox tool inputs and outputs |
| Improvement | Agent optimiser (Preview) proposes better instructions, skills, tool descriptions and model choices |

## Design decisions and trade-offs

A short design-review checklist:

1. **Problem fit**: could a direct model call, RAG or a workflow do this?
2. **Scope**: one domain per agent? Clear definition of done?
3. **Tools**: minimal set, clear descriptions, decision rules, approvals for side effects?
4. **Knowledge**: grounded via Foundry IQ or search, with permissions?
5. **Identity**: per-agent identity, least privilege, OBO for user data?
6. **Limits**: iterations, time-outs, token and cost budgets?
7. **Failure**: graceful degradation, human fallback, error surfacing?
8. **Evaluation**: scenario set, safety tests, CI gate, online monitoring?
9. **Observability**: traces for every tool call, per-agent cost metrics?
10. **Lifecycle**: versioning, pinned production version, model retirement plan?

## Hands-on

Refactor an over-engineered design.

1. Start with a deliberately poor design: five agents (greeter, classifier, policy, order, closer) in a sequential chain, each with all 20 tools.
2. Measure on 30 scenarios: task success (evaluator), latency, tokens per task.
3. Refactor to **one agent** with 6 focused tools, decision rules in instructions, `tool_choice="required"` for order questions, and an approval on refunds.
4. Re-measure. Then add a **maker-checker** reviewer only for refund decisions and measure again.
5. Write a one-page design decision record: which pattern, why, trade-offs, metrics.

## Security, cost and reliability notes

- **OWASP-style agent risks**: prompt injection (direct and indirect), excessive agency, insecure tool output handling, sensitive data disclosure, supply chain (third-party MCP servers). Map each to a mitigation.
- **Blast radius**: separate agents by data sensitivity; one compromised agent shouldn't reach everything.
- **Budgets**: alert on cost per conversation and per agent; cap reasoning effort and iterations.
- **Resilience**: multi-region model deployments behind a gateway; fallback models for degraded operation ([Topic 15 file 13](../15-foundry-models-and-azure-openai/13-rate-limits-quotas-and-resilience.md)).

## Common mistakes

| Mistake | Fix |
|---|---|
| Choosing the pattern before understanding the task | Start at the lowest complexity level |
| Optimising prompts without traces | Read traces; fix tool choice and context first |
| Treating agent output as final | Validate and, where needed, review |
| Measuring only success rate | Also latency, cost, safety and user satisfaction |
| No owner for each agent | Assign owners; register agents for governance |

## Check yourself

1. What are the three levels of agent complexity, and which is often the right enterprise default?
2. Name five patterns that work well and when to use each.
3. List four tool design best practices.
4. Give four reliability practices from the Architecture Center.
5. Name six agent anti-patterns.
6. How do you control cost in multi-agent systems?
7. What does `tool_choice` do, and what are its values?
8. What questions belong in an agent design review?

<details>
<summary>Answers</summary>

1. Direct model call, single agent with tools, multi-agent orchestration; single agent with tools is often the right default.
2. Examples: single focused agent (one domain), triage + specialists (routing), maker-checker (quality-critical outputs), plan-then-execute (complex tasks), human-in-the-loop (consequential actions), deterministic shell with agentic core (mixed processes).
3. Any four: describe purpose and when to use; decision rules for overlapping tools; task-shaped tools; return minimal fields; instruct on failures; treat outputs as untrusted; use `tool_choice`.
4. Any four: time-outs and retries, graceful degradation, surfacing errors, validating outputs before hand-off, circuit breakers, isolation of dependencies, checkpoints.
5. Any six from the anti-pattern table.
6. Right-size models per agent, monitor tokens per agent and run, compact context between agents, limit iterations and reasoning effort, cache where possible.
7. Controls whether the model calls tools: `auto` (model decides), `required` (must call), `none` (no tool calls).
8. Problem fit, scope, tools, knowledge, identity, limits, failure handling, evaluation, observability and lifecycle.

</details>

## References

- [AI agent orchestration patterns (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Single-agent or multi-agent systems (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/single-agent-multiple-agents)
- [Tool best practices for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/concepts/tool-best-practice)
- [Agent optimiser overview (Preview)](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-optimizer-overview)
- [AI agent adoption: business strategy and plan (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
