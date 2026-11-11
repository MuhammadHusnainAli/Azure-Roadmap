# Well-Architected for AI Workloads

## Why it matters

AI workloads break assumptions that traditional architecture relies on. Outputs are non-deterministic, quality can drift without code changes, models get retired on vendor schedules, capacity is quota-bound and expensive, data flows into prompts in ways that create new security and privacy risks, and "correctness" has to be measured statistically. Microsoft extends the Well-Architected Framework with **AI workload guidance**: a design methodology, AI-specific design principles per pillar, an architecture pattern, workload personas, technical design areas (application design, application platform, training and grounding data, data platform) and operational design areas (workload operations, MLOps/GenAIOps, testing and evaluation, responsible AI). It's the frame for every other file in this topic.

## Key concepts

### What makes an AI workload different

| Characteristic | Architectural implication |
|---|---|
| Non-deterministic outputs | Evaluation-driven design, guardrails, human oversight |
| Data-dependent behaviour | Grounding and training data are first-class architecture components |
| Model lifecycle owned by providers | Version pinning, upgrade testing, multi-model portability |
| Capacity and quotas | Capacity planning, gateways, fallbacks (file 07, file 09) |
| New attack surface | Prompt injection, data exfiltration, tool abuse (Topic 23) |
| Cost per token/inference | Cost modelling and controls (Topic 29 file 06) |
| Continuous experimentation | MLOps/GenAIOps pipelines (Topic 22, Topic 27 file 06) |

### Design principles by pillar (summary of WAF AI guidance)

| Pillar | AI-specific principles |
|---|---|
| **Reliability** | Mitigate single points of failure in orchestrators, models and data stores; design for throttling and capacity limits; plan for model version changes; keep retrieval and agent state recoverable |
| **Security** | Protect data used for training and grounding; enforce access at retrieval (security trimming); defend against prompt injection and jailbreaks; least-privilege tools and agent identities; audit model and data lineage |
| **Cost Optimisation** | Choose the smallest model that meets quality; optimise tokens (prompt design, caching, batch); right-size search and compute; monitor cost per outcome |
| **Operational Excellence** | Evaluation-driven development; versioning of prompts, models, indexes and agents; automated pipelines; tracing and observability; safe deployment with canaries |
| **Performance Efficiency** | Latency budgets across retrieval, model and tools; streaming; capacity planning (PTU vs PAYG); performance testing with realistic prompts |

### Design areas

| Area | Focus | Roadmap topics |
|---|---|---|
| **Application design** | Orchestration, agents, tool use, conversation state, UX patterns | 18, 19, 34/04 |
| **Application platform** | Hosting for orchestrators and models, model hosting choices, gateways | 14, 15, 34/05 |
| **Training data design** | Data for fine-tuning and ML training: quality, lineage, privacy | 16, 21 |
| **Grounding data design** | Ingestion, chunking, enrichment, indexing, freshness, security trimming | 17, 34/03 |
| **Data platform** | Lakehouse and pipelines feeding AI | 12 |
| **Workload operations** | Monitoring, incident response, capacity | 26/09 |
| **MLOps and GenAIOps** | Lifecycle automation | 21, 22, 27/06 |
| **Testing and evaluation** | Quality, safety, performance, red teaming | 22 |
| **Responsible AI** | Principles, impact assessment, transparency, oversight | 23 |

### Architecture pattern

A typical generative AI workload has: client/UI → application/orchestrator (agent) → AI gateway → models; knowledge retrieval (AI Search/Foundry IQ) over a grounding data pipeline; tools/APIs; conversation and memory stores; evaluation and observability; guardrails; identity and network controls.

## How it works on Azure

Applying the methodology:

1. **Define the use case and success metrics** (Topic 13 file 09, Topic 22 file 10).
2. **Select the solution approach**: buy, low-code, pro-code (Topic 24 file 05); RAG vs fine-tuning vs agents (Topic 16 file 05).
3. **Design the architecture** from baselines (file 02 onwards), applying WAF AI principles per pillar.
4. **Define evaluation** datasets, metrics and thresholds before building.
5. **Build with GenAIOps**: versioned artefacts, CI gates, tracing.
6. **Operate**: continuous evaluation, cost and capacity monitoring, model lifecycle management.
7. **Assess** with the WAF AI assessment and iterate.

AI design review checklist (excerpt):

| Check | Pillar |
|---|---|
| Fallback when the primary model is throttled or unavailable | Reliability |
| Retrieval enforces user permissions | Security |
| Prompt Shields and agent guardrails configured; tools least-privilege | Security |
| Token budget per request and per tenant; caching | Cost |
| Evaluation datasets and CI gates exist; tracing enabled | Operational Excellence |
| Latency budget defined and load-tested | Performance |
| Impact assessment and human oversight defined | Responsible AI |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Model choice** | Quality vs cost vs latency vs residency; evaluate on your data. |
| **Determinism** | Use structured outputs, lower temperature where supported, deterministic code for critical logic. |
| **Data freshness vs cost** | Incremental indexing; freshness requirements per source. |
| **Autonomy level** | More agent autonomy increases value and risk; add approvals. |
| **Platform vs workload** | Shared AI platform (gateway, guardrails, observability) with workload teams owning apps (file 12). |

## Hands-on

1. Run the WAF AI assessment for an AI workload.
2. Map its components to the design areas and identify gaps.
3. Write AI-specific requirements per pillar (Topic 33 file 03 format).
4. Complete the design review checklist with evidence.
5. Create an ADR for model selection with evaluation evidence.

## Security, cost and reliability notes

- **AI security** extends classic security; include threat modelling for prompts, retrieval and tools.
- **Cost volatility**: usage-based AI costs need budgets, limits and monitoring.
- **Reliability**: model capacity and retirement are external dependencies; design for them.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating AI as a normal API call | Apply AI design areas |
| No evaluation until after launch | Evaluation-driven design |
| Single model with no fallback | Multi-deployment strategy |
| Grounding data treated as an afterthought | Design the data pipeline |
| Ignoring responsible AI until review | Integrate from the start |

## Check yourself

1. What characteristics make AI workloads different?
2. Give one AI-specific principle per pillar.
3. List the technical and operational design areas.
4. Describe the typical generative AI architecture pattern.
5. What are the steps of the design methodology?
6. Why is grounding data design an architecture concern?

<details>
<summary>Answers</summary>

1. Non-determinism, data-dependent behaviour, provider-owned model lifecycle, capacity/quota limits, new attack surface, token costs, continuous experimentation.
2. Reliability: plan for throttling and model changes; Security: security trimming and injection defence; Cost: smallest adequate model and token optimisation; Operational Excellence: evaluation-driven, versioned lifecycle; Performance: latency budgets and capacity planning.
3. Technical: application design, application platform, training data design, grounding data design, data platform. Operational: workload operations, MLOps/GenAIOps, testing and evaluation, responsible AI.
4. Client → orchestrator/agent → AI gateway → models, with retrieval over a grounding pipeline, tools, state stores, evaluation, observability, guardrails, identity and network controls.
5. Define use case and metrics, select approach, design from baselines, define evaluation, build with GenAIOps, operate, assess and iterate.
6. Retrieval quality, freshness and permissions directly determine answer quality and security.

</details>

## References

- [AI workloads on Azure (Well-Architected)](https://learn.microsoft.com/azure/well-architected/ai/)
- [Design principles for AI workloads](https://learn.microsoft.com/azure/well-architected/ai/design-principles)
- [Application design for AI](https://learn.microsoft.com/azure/well-architected/ai/application-design)
- [Grounding data design](https://learn.microsoft.com/azure/well-architected/ai/grounding-data-design)
- [MLOps and GenAIOps](https://learn.microsoft.com/azure/well-architected/ai/mlops-genaiops)
- [Responsible AI](https://learn.microsoft.com/azure/well-architected/ai/responsible-ai)
