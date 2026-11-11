# AI Platform Operating Model

## Why it matters

As AI use spreads, organisations hit the same scaling problems: every team builds its own model access, guardrails, evaluation harness and monitoring; costs are invisible; governance is inconsistent; and scarce AI experts are spread thin. The answer is an **operating model** that defines what's shared (a governed AI platform), what's owned by workload teams (applications and agents), and how expertise, standards and decisions flow (an **AI Centre of Excellence**). Microsoft's Cloud Adoption Framework AI guidance describes establishing an AI CoE, platform architectures and governance; this file turns them into a practical model for architects and consultants advising organisations.

## Key concepts

### Roles and structures

| Body | Responsibilities |
|---|---|
| **Executive sponsor / AI steering group** | Strategy, funding, priorities, risk appetite |
| **AI Centre of Excellence (CoE)** | Standards, reusable patterns, enablement and training, use-case intake support, evaluation methods, vendor/model evaluation, community of practice; evolves from centralised delivery to enabling federated teams |
| **AI platform team** | Builds and runs shared AI services: Foundry resources/hubs of projects, AI gateway, model deployments and capacity (PTUs), guardrail baselines, evaluation and tracing infrastructure, knowledge platform, MCP/tool catalogue, landing zone integration |
| **Workload (product) teams** | Build and operate AI applications and agents; own prompts, agents, data pipelines, evaluation datasets, SLOs and costs |
| **Responsible AI / governance board** | Policies, impact assessments, high-risk approvals (Topic 23 file 09) |
| **Security, privacy, legal** | Threat models, data protection, regulatory compliance |
| **Data platform team** | Grounding data, lakehouse, quality and governance |

### Shared services vs workload responsibilities

| Capability | Platform (shared) | Workload |
|---|---|---|
| Model access | AI gateway, model catalogue/approval, deployments, quotas, PTU pools | Model choice within approved list, prompts |
| Safety | Default guardrails, Prompt Shields, Defender for AI, red teaming tooling | Use-case-specific guardrails, red teaming runs |
| Knowledge | Search/Foundry IQ platform, connectors, ingestion patterns | Domain indexes, chunking choices, permissions mapping |
| Tools | API Center catalogue, MCP hosting patterns, APIM | Tool implementations owned by system owners |
| Observability | Tracing, Application Insights, dashboards, alerts templates | Workload SLOs, dashboards, on-call |
| Evaluation | Evaluation pipelines, judge models, templates | Datasets, thresholds, results |
| Identity and governance | Entra Agent ID patterns, Control Plane, Agent 365, policies | Agent registration, owners, access requests |
| Cost | Chargeback/showback, budgets | Cost per outcome, optimisation |
| Enablement | Training, templates, golden paths | Adoption and feedback |

### Funding and chargeback

Options: centrally funded platform (simple, encourages adoption), consumption-based chargeback (tokens and services per team via gateway metrics, Topic 29 file 04), or hybrid (platform base funded centrally, consumption charged back).

### Maturity evolution

| Stage | Characteristics |
|---|---|
| **Experiment** | Isolated PoCs, direct model access, little governance |
| **Standardise** | CoE formed, approved models, gateway, basic guardrails, intake process |
| **Scale** | Platform with self-service, golden paths, evaluation pipelines, chargeback, agent registry |
| **Optimise** | Federated teams, continuous evaluation, portfolio management of AI value, automated governance |

### Metrics

Number of AI apps/agents in production, time from idea to production, platform adoption (% of AI traffic through gateway), evaluation coverage, incidents, cost per outcome, business value delivered (Topic 22 file 10), skills/training completion.

## How it works on Azure

Reference platform layout (landing zone aligned):

```text
Platform subscription(s): AI gateway (APIM multi-region), shared Foundry resource(s) with projects per team or per app,
  PTU deployments in pooled capacity, central Application Insights/Log Analytics, Defender for AI, Purview,
  Foundry Control Plane, API Center (tools/MCP catalogue), evaluation pipelines and red teaming automation
Workload subscriptions: app hosting, workload Foundry projects (or own Foundry resources where isolation required),
  AI Search/Foundry IQ knowledge bases, data pipelines, agents; consume models only via the gateway
Governance: management group policies (approved regions/models, private networking, local auth disabled),
  Agent 365 registry, intake and review process, chargeback reports
Enablement: golden-path templates (agent repo + IaC + CI with evaluation gates), training, office hours
```

Project and resource strategy: shared Foundry resource with **projects per team/application** for most teams (shared connections and governance, per-project RBAC), and dedicated Foundry resources for workloads needing strict isolation, separate billing or specific regions ([Topic 14 file 02](../14-microsoft-foundry-platform/02-foundry-resources-and-projects.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Centralised vs federated** | Start centralised to establish standards; federate delivery as maturity grows; keep platform and policy central. |
| **Mandatory gateway** | Require AI traffic through the gateway for visibility, limits and failover. |
| **Approved models list** | Curated, evaluated models with a fast path for adding new ones. |
| **Platform as product** | Roadmap, SLAs, documentation and feedback loops (Topic 27 file 10). |
| **Avoid bottlenecks** | CoE enables; it shouldn't approve every prompt change. Proportional governance by risk tier. |

## Hands-on

1. Draft an AI operating model for an organisation: bodies, responsibilities, decision rights.
2. Fill the shared vs workload responsibility table for your context.
3. Design the platform layout (subscriptions, Foundry resources/projects, gateway, catalogues).
4. Choose a funding/chargeback model and build a showback mock-up from gateway metrics.
5. Assess maturity and define the next three platform capabilities to build.

## Security, cost and reliability notes

- **Shared platform = shared blast radius**: zone/multi-region gateway, change management, capacity monitoring.
- **Least privilege** between teams on shared resources (project-level RBAC).
- **Cost transparency** drives responsible usage.
- **Platform reliability** becomes critical once many products depend on it.

## Common mistakes

| Mistake | Fix |
|---|---|
| Every team building its own AI stack | Shared platform and golden paths |
| CoE as a gatekeeping bottleneck | Enablement and risk-proportional governance |
| No chargeback or visibility | Gateway metrics and showback |
| Shared resources without isolation controls | Projects, RBAC, quotas |
| Platform without product management | Roadmap, SLAs, feedback |

## Check yourself

1. What are the responsibilities of an AI CoE?
2. Split responsibilities between the AI platform and workload teams.
3. Describe three funding models.
4. What are the maturity stages?
5. When should a workload get its own Foundry resource?
6. Which metrics show an AI platform is working?

<details>
<summary>Answers</summary>

1. Standards, reusable patterns, enablement and training, intake support, evaluation methods, model/vendor evaluation, community of practice.
2. Platform: gateway, model deployments and quotas, default guardrails, knowledge platform, tool catalogue, observability and evaluation infrastructure, governance tooling, chargeback. Workload: apps, agents, prompts, domain indexes, datasets, SLOs, costs.
3. Central funding, consumption chargeback, hybrid base plus consumption.
4. Experiment, standardise, scale, optimise.
5. When strict isolation, separate billing, specific regions or compliance boundaries are required.
6. AI apps in production, idea-to-production time, gateway adoption, evaluation coverage, incidents, cost per outcome, business value, training completion.

</details>

## References

- [Establish an AI Center of Excellence (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/center-of-excellence)
- [AI architecture guidance for Azure PaaS (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/platform/architectures)
- [AI adoption (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/)
- [Foundry Control Plane](https://learn.microsoft.com/azure/foundry/control-plane/overview)
