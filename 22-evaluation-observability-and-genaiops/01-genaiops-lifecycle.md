# GenAIOps Lifecycle

## Why it matters

Generative AI applications fail differently from traditional software and differently from classical ML. There's no single "model accuracy" number; behaviour depends on prompts, retrieved data, tools, model versions and user input; and quality can degrade without a single line of code changing (a model upgrade, an index refresh, a new kind of user question). **GenAIOps** (also called LLMOps or AgentOps) is the set of practices that takes AI apps and agents from experiment to production and keeps them healthy: evaluation-driven development, tracing, gated releases, production monitoring and continuous improvement. Every role needs the mental model; engineers and architects need the mechanics.

## Key concepts

### How GenAIOps differs from MLOps

| | MLOps (Topic 21) | GenAIOps |
|---|---|---|
| Main asset | A trained model | An application/agent: model + prompts + retrieval + tools + orchestration |
| Typical change | Retrain with new data | Change prompt, tool, index, model version or framework |
| Quality measure | Accuracy, AUC, RMSE on labelled data | Groundedness, relevance, task completion, safety, often judged by LLMs or humans |
| Data pipelines | Feature engineering | Plus chunking, embedding and **index maintenance** for RAG |
| Risks | Drift, bias | Plus hallucination, prompt injection, unsafe content, tool misuse, cost spikes |
| Non-determinism | Low | High; same input can produce different outputs |

Microsoft's Architecture Center guidance: reuse MLOps investments (DataOps, pipelines, registries, CI/CD) and extend them for prompting, RAG and agents, across an **inner loop** (DataOps, experimentation, evaluation) and an **outer loop** (deployment, inferencing, monitoring).

### Lifecycle stages

| Stage | Activities | Foundry capabilities |
|---|---|---|
| **1. Ideate and qualify** | Use case, success metrics, risks, data readiness | [Topic 13 file 09](../13-ai-and-generative-ai-foundations/09-ai-use-case-qualification.md) |
| **2. Base model selection** | Compare quality, latency, cost, safety | Model leaderboards, benchmark evaluations ([Topic 15 file 06](../15-foundry-models-and-azure-openai/06-model-router-and-model-selection.md)) |
| **3. Build and experiment** | Prompts, RAG, tools, agents; inner-loop evaluation | Playgrounds, Foundry Toolkit, Agent Framework, tracing |
| **4. Pre-production evaluation** | Test datasets, simulated conversations, safety and red teaming, CI/CD gates | Evaluators, cloud evaluation, AI Red Teaming Agent, GitHub Actions/Azure DevOps |
| **5. Deploy** | Versioned agents/apps, safe rollout | Agent versions and endpoints, gateways ([Topic 18 file 12](../18-agents-with-foundry-agent-service/12-publishing-agents.md)) |
| **6. Operate and monitor** | Traces, operational metrics, continuous and scheduled evaluation, scheduled red teaming, alerts | Foundry observability dashboard, Application Insights, Azure Monitor |
| **7. Improve** | Turn production failures into test cases; optimise prompts, tools, models | Traces → datasets, cluster analysis, agent optimiser |

Microsoft's observability framing groups evaluation across **base model selection**, **pre-production evaluation** and **post-production monitoring**, supported by three capabilities: **evaluation**, **monitoring** and **tracing**.

### Artefacts to version

| Artefact | Where |
|---|---|
| Prompts, agent definitions, skills | Git; Foundry agent versions ([Topic 16 file 04](../16-prompt-engineering-and-model-customisation/04-prompt-management.md)) |
| Tool definitions, MCP servers, toolboxes | Git, API Center, Foundry toolbox versions |
| Model deployments and versions | IaC; deployment upgrade policies |
| Indexes and chunking configuration | IaC/definitions; index aliases |
| Evaluation datasets and evaluator configs | Git and Foundry datasets |
| Guardrail (content filter) configurations | IaC |
| Infrastructure | Bicep/Terraform |

### Roles

| Role | Responsibility |
|---|---|
| AI engineer | Build agent/app, instrument tracing, write evaluations |
| Data/knowledge engineer | Ingestion, chunking, index freshness |
| Platform/MLOps engineer | Environments, CI/CD, gateways, monitoring |
| Product owner / SME | Success criteria, golden datasets, human review |
| Responsible AI / security | Risk assessment, red teaming, guardrails, sign-off |
| Operations / SRE | Alerts, incidents, capacity and cost |

## How it works on Azure

```text
Dev (Foundry project) ──► CI: unit tests + eval smoke set ──► Test project: full eval + red teaming ──► approval
       ▲                                                                                              │
       │                                                                                              ▼
  improve (optimiser, new tests) ◄── traces, feedback, continuous eval, alerts ◄── Prod project (pinned versions, gateway)
```

Typical Azure building blocks: separate Foundry projects (or resources) per environment, Application Insights per environment, AI gateway (API Management) in front of models, GitHub Actions/Azure DevOps for gates, Azure Monitor alerts, Azure Policy and Defender for Cloud for governance and security ([Topic 23](../23-responsible-ai-security-and-governance/README.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Evaluation-driven development** | Define metrics and datasets before building; every change is measured. |
| **Environments** | At least dev and prod; regulated workloads add test/staging with full evaluation and red teaming. |
| **Gates** | Automate smoke evaluation on every relevant pull request; full evaluation and safety scans before production; human sign-off for high-risk apps. |
| **Model upgrades** | Treat as releases: pin versions, evaluate new versions, roll out gradually ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)). |
| **Feedback loop** | Collect user feedback and traces; turn failures into test cases weekly. |
| **Ownership** | One accountable owner per agent; registry/inventory of agents ([Topic 23 file 09](../23-responsible-ai-security-and-governance/09-ai-governance-operating-model.md)). |

## Hands-on

1. Map an existing or planned agent onto the seven lifecycle stages; list the artefacts and owners for each.
2. Create a repository layout: `agents/`, `prompts/`, `tools/`, `eval/datasets/`, `eval/config/`, `infra/`, `.github/workflows/`.
3. Write a one-page "definition of done" for production: metrics and thresholds, safety scan results, tracing enabled, alerts configured, rollback plan.
4. Identify three ways quality could degrade without a code change (model update, index change, new user behaviour) and the monitoring signal that would catch each.

## Security, cost and reliability notes

- **Change control** extends beyond code: prompts, models, indexes and filters all need versioning and review.
- **Data in traces and evaluations** may include user content; apply access controls and retention ([file 06](06-tracing-with-opentelemetry.md)).
- **Cost observability**: track tokens, tool calls and evaluation spend per agent and environment.
- **Reliability**: rollback paths for each artefact type (agent version, deployment, index alias).

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating GenAI apps like classic software with unit tests only | Add evaluation datasets and evaluators |
| Treating them like classic ML with one accuracy metric | Measure quality, safety, task completion and cost |
| Shipping without tracing | Instrument from day one |
| No owner or inventory | Register agents with owners |
| Never updating test sets | Add production failures continuously |

## Check yourself

1. How does GenAIOps differ from MLOps?
2. What are the inner and outer loops?
3. List the lifecycle stages from qualification to improvement.
4. Which artefacts must be versioned in a GenAI application?
5. What three observability capabilities does Foundry provide?
6. How can quality degrade without a code change?

<details>
<summary>Answers</summary>

1. The asset is an application of model + prompts + retrieval + tools; changes are prompt/tool/index/model updates; quality is judged with groundedness, relevance, task completion and safety, often by LLM or human judges; new risks like injection and hallucination.
2. Inner: DataOps, experimentation, evaluation; outer: deployment, inferencing and monitoring.
3. Ideate/qualify, base model selection, build/experiment, pre-production evaluation, deploy, operate/monitor, improve.
4. Prompts and agent definitions, tools/MCP/toolboxes, model deployments, indexes and chunking config, evaluation datasets and configs, guardrails, infrastructure.
5. Evaluation, monitoring and tracing.
6. Model version updates, index or data changes, new user behaviour or question types, tool/API changes.

</details>

## References

- [Observability in generative AI (Foundry)](https://learn.microsoft.com/azure/foundry/concepts/observability)
- [Evaluation in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/evaluation-overview)
- [GenAIOps for organisations with MLOps investments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/genaiops-for-mlops)
- [MLOps and GenAIOps for AI workloads (Well-Architected)](https://learn.microsoft.com/azure/well-architected/ai/mlops-genaiops)
- [Integrate, manage and operate AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/integrate-manage-operate)
