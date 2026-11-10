# AI Solution Patterns

## Why it matters

Most generative AI solutions are variations of a small number of patterns: **chat**, **retrieval-augmented generation (RAG)**, **extraction**, **classification**, **summarisation and generation**, **copilots**, **single agents** and **multi-agent systems**. Naming the pattern early tells you what data, controls, evaluation and cost profile the solution needs, and stops teams from building an autonomous multi-agent system when a well-designed RAG app would do. This file is the map that topics 16–24 fill in.

## Key concepts

### The pattern spectrum

```text
More deterministic, cheaper, easier to evaluate  ◄──────────────────────────►  More autonomous, flexible, harder to control

Classification · Extraction · Summarisation · Chat · RAG · Copilot · Single agent · Workflow of agents · Multi-agent (autonomous)
```

**Microsoft's guidance is to use the simplest pattern that meets the requirement**, and add autonomy only where it adds clear value.

### Pattern catalogue

| Pattern | What it does | Key components | Typical risks | Evaluate with |
|---|---|---|---|---|
| **Classification / routing** | Assign a label (intent, category, sentiment, priority) | Prompt with label set, structured output, small model; or fine-tuned SLM | Label drift, ambiguous classes | Accuracy, precision/recall per class |
| **Extraction** | Turn unstructured input into structured fields | Content Understanding or Document Intelligence, or LLM + JSON schema (structured outputs) | Missing or invented fields | Field-level accuracy, schema validity |
| **Summarisation / transformation** | Summarise, translate, rewrite, draft | Prompt templates, long context or chunked map-reduce | Omissions, distortion | Coverage, faithfulness, human rating |
| **Chat (ungrounded)** | Conversational assistant using model knowledge | Chat model, system prompt, history management | Hallucination, off-topic, outdated knowledge | Relevance, safety |
| **RAG** | Answer from your content with citations | Ingestion, chunking, embeddings, AI Search (hybrid + semantic ranker), knowledge base (Foundry IQ), prompt with citations | Retrieval misses, permission leaks, stale content | Retrieval recall, groundedness, citation accuracy |
| **Copilot** | Assistant embedded in an app or workflow that helps a user do their job | RAG + app context + actions with user confirmation | Over-trust, actions without review | Task completion, user satisfaction, error rate |
| **Single agent** | A model that plans and calls tools in a loop to achieve a goal | Agent runtime (Foundry Agent Service), tools (functions, MCP, OpenAPI, search, code interpreter), memory, guardrails | Wrong tool calls, prompt injection, runaway loops, cost | Task success, tool-call accuracy, safety, cost per task |
| **Agentic workflow** | Deterministic workflow with agent steps (some steps reason, others are fixed) | Foundry workflow agents, Microsoft Agent Framework workflows, Logic Apps agent loops, Durable Functions | Complexity across steps | End-to-end success, step-level metrics |
| **Multi-agent system** | Several specialised agents collaborating (orchestrator and workers, handoff, group chat) | Orchestration patterns, agent-to-agent protocols (A2A), shared state | Coordination failures, compounding errors, latency and cost multiplication, harder debugging | Task success, trace analysis, cost |

### RAG in one diagram

```text
Ingest:  documents → extract → chunk → embed → index (AI Search, with ACLs and metadata)
Query:   user question → (rewrite) → hybrid retrieve + rerank → top passages
         → prompt: instructions + passages + question → model → answer with citations
```

RAG solves "the model doesn't know our data" and "the model's knowledge is stale", and it enables **citations** and **permission trimming**. Topic 17 goes deep.

### What makes something an agent

An **agent** combines:

1. a **model** that reasons and decides;
2. **instructions** (goal, persona, rules);
3. **tools** it can call (search, APIs, code, other agents);
4. **memory or state** across turns;
5. a **loop**: observe → think → act → observe, until done;
6. **guardrails**: limits, approvals, content safety, identity.

If the steps are always the same, you don't need an agent; use a workflow. Agents earn their cost when the path depends on the situation.

### Multi-agent orchestration patterns

| Pattern | Description | Use when |
|---|---|---|
| **Sequential** | Agents run in a fixed order, each refining the output | Clear pipeline (draft → review → format) |
| **Concurrent** | Agents work in parallel; results aggregated | Independent perspectives (legal, finance, risk review) |
| **Handoff** | One agent transfers control to a more suitable agent | Triage to specialists |
| **Group chat / debate** | Agents converse, often with a manager | Brainstorming, critique |
| **Magentic / planner-orchestrator** | A manager plans dynamically and delegates to workers | Open-ended, complex tasks |

These are the patterns implemented by the Microsoft Agent Framework and described in the Azure Architecture Center (topic 19).

### Human-in-the-loop levels

| Level | Example |
|---|---|
| **Human does, AI suggests** | Copilot drafts an email; human edits and sends |
| **AI does, human approves** | Agent prepares a refund; human approves the action |
| **AI does, human monitors** | Agent triages tickets; humans review samples and exceptions |
| **Fully autonomous** | Only for low-risk, reversible, well-evaluated actions |

## How it works on Azure

| Pattern | Primary Azure building blocks |
|---|---|
| Classification, extraction, summarisation | Foundry Models (structured outputs), Content Understanding, Document Intelligence, batch deployments |
| Chat | Foundry Models, Responses API, App Service / Container Apps front end |
| RAG | Azure AI Search + **Foundry IQ** knowledge bases, Foundry Models, Blob/ADLS/SharePoint sources |
| Copilot in your app | Foundry Agent Service or Responses API with tools; or extend **Microsoft 365 Copilot** with agents built in **Copilot Studio** |
| Single agent | **Foundry Agent Service** (prompt agents), tools via MCP/OpenAPI/Functions, Foundry IQ, Fabric data agents |
| Agentic workflow | Foundry **workflow agents**, **Microsoft Agent Framework** workflows, Logic Apps agent loops, Durable Functions |
| Multi-agent | Foundry Agent Service (connected agents, workflows, hosted agents), Microsoft Agent Framework, A2A |
| Governance across all | Foundry Control Plane, AI gateway (APIM), Entra Agent ID, Purview, Content Safety, evaluations |

### Example: choosing the pattern

| Request | Pattern |
|---|---|
| "Route incoming emails to the right team" | Classification (SLM or small model, structured output) |
| "Pull totals and dates from supplier invoices into SAP" | Extraction (Content Understanding) + integration (Logic Apps) |
| "Answer staff questions about HR policies with sources" | RAG |
| "Help claims handlers draft decisions inside our claims system" | Copilot (RAG + context + suggested actions) |
| "Resolve password-reset tickets end to end" | Single agent with tools and guardrails |
| "Produce a due-diligence report from legal, financial and market analyses" | Multi-agent (concurrent specialists + aggregator) or agentic workflow |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Workflow vs agent | Deterministic workflow when steps are known; agent when the path depends on the input. Hybrid "workflow with agent steps" is often best |
| Single vs multi-agent | Start with one agent and good tools. Split into multiple agents only for clear separation of concerns, different permissions, different models, or team ownership boundaries |
| Build vs buy | Microsoft 365 Copilot and Copilot Studio for productivity scenarios over Microsoft 365 data; Foundry for custom apps, external users, complex integration (topic 24) |
| Autonomy level | Match to risk: reversible and low-impact actions can be autonomous; financial, legal or customer-facing actions need approval |
| Evaluation | Define success metrics per pattern before building (topic 22) |

## Hands-on

1. For five real or invented business requests, name the pattern, the data needed, the human-in-the-loop level and two key risks.
2. Implement the simplest version of one pattern (for example classification with structured output) and an evaluation set of 30 examples.
3. In the Foundry portal, create a prompt agent with one tool (for example a knowledge base or code interpreter) and inspect the trace of its tool calls.
4. Redesign a proposed "multi-agent" solution as a single agent or workflow and compare complexity.

## Security, cost and reliability notes

- **Security:** risk grows along the spectrum. Agents with write tools need least-privilege identities (Entra Agent ID), approvals and prompt-injection defences.
- **Cost:** agents and multi-agent systems multiply model calls per task (planning, tool calls, reflection). Measure **cost per completed task**, not per call.
- **Reliability:** each extra autonomous step compounds error rates. Prefer deterministic steps where possible and add checkpoints.

## Common mistakes

- Starting with a multi-agent architecture because it's fashionable.
- Calling a fixed workflow an "agent" and paying for unnecessary reasoning.
- RAG without permission trimming or evaluation.
- Copilots that act without user confirmation.
- No definition of success before building.

## Check yourself

1. List the main generative AI solution patterns from most deterministic to most autonomous.
2. What problems does RAG solve?
3. What are the components of an agent?
4. When is a workflow better than an agent?
5. Name four multi-agent orchestration patterns.
6. How would you decide the human-in-the-loop level?

<details>
<summary>Answers</summary>

1. Classification, extraction, summarisation/generation, chat, RAG, copilot, single agent, agentic workflow, multi-agent system.
2. Lack of knowledge of private data, stale knowledge, need for citations, and permission-aware answers.
3. A model, instructions, tools, memory or state, an observe-think-act loop, and guardrails.
4. When the steps are known and stable: workflows are cheaper, more predictable and easier to test.
5. Sequential, concurrent, handoff, group chat (also magentic/planner-orchestrator).
6. By the impact and reversibility of actions, regulatory requirements, and measured reliability: higher risk needs approval or review.

</details>

## References

- [AI agent orchestration patterns (Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [RAG solution design and evaluation guide](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
- [What is Foundry Agent Service?](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [AI adoption: plan and govern (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/)
