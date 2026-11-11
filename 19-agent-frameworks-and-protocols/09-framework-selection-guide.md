# Framework Selection Guide

## Why it matters

There are now many ways to build an agent on Microsoft's stack and beyond: Foundry prompt agents, Foundry hosted agents, Microsoft Agent Framework, Copilot Studio, the Microsoft 365 Agents SDK, LangGraph, CrewAI, the OpenAI Agents SDK, the Claude Agent SDK, or no framework at all. Picking the wrong one costs months: rewrites when a framework is deprecated (AutoGen), lock-in where portability mattered, or heavy code where configuration would do. This guide gives a decision process and a comparison you can use with engineering teams and clients.

## Key concepts

### First question: do you need a framework?

| Situation | Answer |
|---|---|
| Single model call (classify, summarise, extract) | **No framework**: call the Responses API with structured outputs |
| Fixed RAG Q&A | **No framework** or a thin library: Azure AI Search + Responses API, or Foundry IQ |
| One agent with a few tools, standard behaviour | **Foundry prompt agent** (configuration, no orchestration code) |
| Agent logic must live in your app code but no complex orchestration | **Responses API "ephemeral agent"** pattern ([Topic 18 file 02](../18-agents-with-foundry-agent-service/02-foundry-agent-service-overview.md)) |
| Custom loops, middleware, multi-agent workflows, checkpoints | **A framework**: Agent Framework (default on Microsoft stack) or your team's existing framework |

Frameworks add abstractions, dependencies and upgrade work. Use one when its orchestration, state, tooling or ecosystem saves more than it costs.

### The options

| Option | Type | Strengths | Watch-outs |
|---|---|---|---|
| **Foundry prompt agent** | Managed, declarative | No code; versioning, endpoint, identity, publishing, evaluation, optimiser | Limited custom logic; Copilot harness for richer workflows is Preview |
| **Foundry hosted agent** | Managed hosting for code | Any framework; sessions, identity, scaling, protocols (Responses, Invocations, A2A) | You maintain the code and container |
| **Microsoft Agent Framework** | Open-source SDK (Python, .NET; Go preview) | Successor to SK and AutoGen; agents, harness, workflows (sequential, concurrent, handoff, group chat, Magentic), checkpoints, middleware, MCP and A2A, many model providers, deep Foundry integration | Fast-moving; pin versions |
| **Copilot Studio** | Low-code SaaS | Makers, Microsoft 365 integration, connectors, governance | Less code-level control ([Topic 24](../Roadmap.md)) |
| **Microsoft 365 Agents SDK / Teams SDK** | Channel SDKs | Teams, Copilot and multi-channel delivery, Activity Protocol, state | Not an AI orchestrator; pair with a brain |
| **LangChain / LangGraph** | Open-source | Large ecosystem; graph state machines; first-class Foundry integration (`langchain-azure-ai`) and hosting | Different abstractions from Microsoft samples |
| **OpenAI Agents SDK** | Open-source | Lightweight, Responses-native, handoffs | Bring-your-own hosting on Foundry; tracing via OpenTelemetry |
| **Claude Agent SDK** | Open-source | Agentic coding and computer-centric workflows with Claude models | Use Claude via Foundry; bring-your-own hosting |
| **CrewAI, LlamaIndex** | Open-source | Role-based crews; data/RAG-centric agents | Bring-your-own hosting; integrate tracing yourself |
| **Semantic Kernel** | Open-source (v1.x supported) | Existing investments | Agent features continue in Agent Framework; plan migration |
| **AutoGen** | **Archived** | — | Don't use for new or production work |

### Decision flow

```text
1. Can configuration do it?            → Foundry prompt agent (or Copilot Studio for makers)
2. Is it mainly a channel/UI problem?  → Microsoft 365 Agents SDK / Teams SDK + a brain
3. Need code-level orchestration?
      a. Existing working framework?   → keep it (LangGraph, CrewAI, …) and host on Foundry
      b. New build on Microsoft stack? → Microsoft Agent Framework
      c. Strong team preference?       → OpenAI Agents SDK / Claude Agent SDK / LangGraph, hosted on Foundry
4. Where does it run?                  → Foundry hosted agent (default) · Durable Functions · Container Apps/AKS
5. How do tools connect?               → MCP via Foundry Toolbox (portable across all of the above)
6. How do agents talk to each other?   → A2A across boundaries; in-process workflows within one codebase
```

### Evaluation criteria

| Criterion | Questions |
|---|---|
| **Team skills** | Python, .NET, TypeScript? Existing framework experience? |
| **Orchestration needs** | Single agent, workflows, multi-agent patterns, human-in-the-loop, long-running? |
| **Hosting and operations** | Managed (Foundry) or self-managed? Scale, sessions, identity, networking? |
| **Model portability** | Must you switch between Foundry, OpenAI, Anthropic, local models? |
| **Governance** | Entra identity, Agent 365 registry, content safety, tracing, evaluation available? |
| **Ecosystem** | Connectors, vector stores, community, samples? |
| **Lifecycle** | Support status, release cadence, migration risk? |
| **Interoperability** | MCP and A2A support? |
| **Cost** | Licences (Copilot Studio, Microsoft 365 Copilot), hosting, token efficiency of the framework's loops |

### Mix and match

Real solutions combine layers:

| Layer | Example choice |
|---|---|
| Channel | Teams via Foundry publishing or Agents SDK |
| Brain | Foundry prompt agent, or Agent Framework workflow as a hosted agent |
| Tools | Foundry Toolbox (MCP), OpenAPI, functions |
| Knowledge | Foundry IQ |
| Inter-agent | A2A |
| Governance | Entra Agent ID, Agent 365, Foundry evaluations and tracing |

## How it works on Azure

Whatever you pick, standardise the platform around it:

- **Models** from Foundry with Entra authentication and an AI gateway for quotas and failover.
- **Tools** in Foundry toolboxes; servers registered in API Center.
- **Hosting** as Foundry hosted agents for identity, sessions and publishing.
- **Observability** through OpenTelemetry into Application Insights; Foundry evaluations on traces.
- **Identity** per agent (Entra Agent ID) with least privilege; OBO for user data.

This keeps frameworks swappable: the expensive platform decisions (identity, tools, knowledge, observability) don't depend on the framework.

## Design decisions and trade-offs

| Trade-off | Guidance |
|---|---|
| **Managed vs code** | Managed (prompt agents) is faster and governed; code (frameworks) is flexible. Start managed; move to code when you hit real limits. |
| **Microsoft framework vs third-party** | Agent Framework has the tightest Foundry, Microsoft 365 and Azure integration and Microsoft support; third-party frameworks may have larger communities or existing code. Host either on Foundry. |
| **Standardise or allow choice** | Recommend a default (usually Agent Framework or prompt agents) but allow exceptions that meet platform standards (MCP tools, OpenTelemetry, Entra identity). |
| **Portability** | MCP for tools and A2A for agents reduce lock-in more than framework choice does. |
| **Maturity** | Prefer GA features for production; isolate preview features behind flags. |

## Hands-on

Run a framework selection workshop for a case study.

1. **Case**: "An IT helpdesk agent in Teams that answers policy questions, checks device status in Intune, opens ServiceNow tickets with approval, and hands complex cases to a human."
2. For each requirement, mark: configuration-only, code needed, channel-specific, multi-agent.
3. Score three candidate designs (1–5) on the evaluation criteria:
   - A: Foundry prompt agent + Foundry IQ + MCP tools + Foundry Teams publishing.
   - B: Agent Framework handoff workflow as a hosted agent + Agents SDK channel.
   - C: Copilot Studio agent + Foundry IQ knowledge + connectors.
4. Choose one, document why the others lost, and list the platform standards (identity, tools, observability) that would let you switch later.
5. Build a thin slice of the chosen design in a day.

## Security, cost and reliability notes

- **Supply chain**: every framework adds dependencies; scan, pin and update; avoid archived projects in production.
- **Consistent controls**: whichever framework, enforce content safety, approvals for side effects, least-privilege identity and tracing.
- **Cost**: frameworks differ in prompts and loop behaviour; measure tokens per task, not just per call.
- **Support**: Microsoft-supported components (Foundry, Agent Framework, Agents SDK) simplify enterprise support cases.

## Common mistakes

| Mistake | Fix |
|---|---|
| Choosing a framework before defining requirements | Start from the decision flow |
| Using a heavy framework for a single model call | Call the API directly |
| Rewriting good LangGraph code to "be Microsoft" | Host it on Foundry |
| Starting new work on AutoGen | Use Agent Framework |
| Confusing channel SDKs with orchestration | Pair the Agents SDK with a brain |
| Letting each team pick tools, identity and telemetry differently | Standardise platform layers |

## Check yourself

1. When do you need no framework at all?
2. What does Foundry offer when configuration is enough?
3. Why is Microsoft Agent Framework the default for new code on the Microsoft stack?
4. When should you keep a third-party framework?
5. What role do the Microsoft 365 Agents SDK and Teams SDK play?
6. Which two protocols reduce lock-in most?
7. List five evaluation criteria for choosing a framework.
8. Which platform layers should be standardised regardless of framework?

<details>
<summary>Answers</summary>

1. Single model calls, fixed RAG Q&A, or simple agents where the Responses API or a prompt agent suffices.
2. Prompt agents: managed runtime, versioning, endpoint, identity, publishing, evaluation and optimisation without code.
3. It succeeds SK and AutoGen, offers agents, harness, workflows, checkpoints, middleware, MCP/A2A, multi-provider models, and the tightest Foundry and Microsoft 365 integration with Microsoft support.
4. When the code works, the team is productive and the framework is maintained; host it on Foundry with protocol SDKs or integrations.
5. They're channel layers for Teams, Microsoft 365 Copilot and other channels; they don't provide the AI brain.
6. MCP (tools) and A2A (agents).
7. Any five: team skills, orchestration needs, hosting/operations, model portability, governance, ecosystem, lifecycle/support, interoperability, cost.
8. Models and gateway, tools (toolboxes/MCP), hosting, identity, observability and evaluation (and knowledge via Foundry IQ).

</details>

## References

- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/)
- [What is Foundry Agent Service? (choose how to build)](https://learn.microsoft.com/azure/foundry/agents/overview)
- [What are hosted agents?](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents)
- [Get started with LangChain and LangGraph in Foundry](https://learn.microsoft.com/azure/foundry/how-to/develop/langchain)
- [What is the Microsoft 365 Agents SDK?](https://learn.microsoft.com/microsoft-365/agents-sdk/agents-sdk-overview)
- [AI agent orchestration patterns](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [AI agent adoption (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
