# Azure Well-Architected Framework

## Why it matters

Every architecture is a set of trade-offs: more resilience costs more money, tighter security adds friction, faster performance may reduce efficiency. Without a shared framework, design reviews become arguments about preferences. The **Azure Well-Architected Framework (WAF)** gives architects, engineers and stakeholders a common vocabulary and set of principles across five pillars (**Reliability, Security, Cost Optimisation, Operational Excellence and Performance Efficiency**), with design principles, checklists, recommendations, trade-off guidance, workload-specific guidance (AI, mission-critical, SaaS, sustainability) and per-service guides. It's the backbone of Azure architecture reviews (file 02), the Architecture Center (file 09) and the AZ-305 exam.

## Key concepts

### The five pillars

| Pillar | Concern | Core principles (summarised) |
|---|---|---|
| **Reliability** | Resiliency, availability, recovery | Design for business requirements, resilience and recovery; keep it simple; design for operations; health modelling; test failures (Topic 28) |
| **Security** | Confidentiality, integrity, availability; threat detection and mitigation | Plan readiness; design to protect confidentiality and integrity; sustain availability; continuously improve; Zero Trust (Topic 25) |
| **Cost Optimisation** | Cost modelling, budgets, reducing waste | Develop cost-management discipline; design with a cost-efficiency mindset; design for usage and rate optimisation; monitor and optimise over time (Topic 29) |
| **Operational Excellence** | Observability, DevOps practices, safe deployment | Embrace DevOps culture; establish development standards; evolve operations with observability; deploy with confidence; automate for efficiency (Topics 26–27) |
| **Performance Efficiency** | Scaling, capacity, performance testing | Negotiate realistic performance targets; design to meet capacity requirements; achieve and sustain performance; improve efficiency through optimisation |

Each pillar provides **design principles**, a **design review checklist**, **recommendation guides** and **trade-offs** with other pillars.

### Trade-offs

| Decision | Benefits | Costs |
|---|---|---|
| Multi-region active-active | Reliability, performance for global users | Cost, operational complexity, data consistency |
| Private networking everywhere | Security | Cost, DNS complexity, developer friction |
| Aggressive autoscale-in | Cost | Performance during spikes, reliability |
| More telemetry | Operational excellence | Cost (ingestion), possibly privacy |
| Caching | Performance, cost | Consistency, complexity |
| Reserved capacity | Cost | Flexibility |

Good architecture makes trade-offs explicit and recorded (file 08), aligned to business priorities.

### Workload guidance and service guides

| Asset | Use |
|---|---|
| **Workload guidance** | AI workloads, mission-critical, SaaS/multitenant, Oracle, SAP, HPC, sustainability, Azure Virtual Desktop and more |
| **Service guides** | Per-service WAF recommendations (App Service, AKS, Cosmos DB, Azure OpenAI/Foundry, Storage, SQL, Front Door, etc.) mapped to the pillars |
| **Assessment** | Azure Well-Architected Review on Microsoft Learn Assessments |
| **Advisor** | Recommendations grouped by pillar categories |
| **Architecture Center** | Reference architectures applying WAF |

### Workload vs platform

WAF focuses on the **workload** (an application and its supporting components owned by a team), complementing the **Cloud Adoption Framework (CAF)**, which focuses on organisational adoption and platform landing zones ([Topic 03](../03-governance-and-landing-zones/README.md)).

## How it works on Azure

Applying WAF during design:

1. **Define the workload**: users, flows, criticality, requirements and constraints (file 03).
2. **Set targets** per pillar: SLOs, RTO/RPO, security requirements, budget, performance targets.
3. **Design** using architecture styles (file 05), patterns (file 04) and service guides.
4. **Evaluate trade-offs** pillar by pillar; record decisions as ADRs.
5. **Review** with the WAF checklist or assessment (file 02).
6. **Operate and iterate**: monitor against targets; re-review after significant changes.

Design review snippet (checklist style) for a web API:

| Pillar | Check | Status |
|---|---|---|
| Reliability | Zone-redundant App Service and SQL; health probes; retry policies | ✅ |
| Security | Private endpoints, managed identity, WAF on Front Door, Defender plans | ⚠ Defender for APIs not enabled |
| Cost | Autoscale rules, savings plan for baseline, budgets | ✅ |
| Operational Excellence | IaC, CI/CD with approvals, Application Insights, runbooks | ⚠ No runbook for failover |
| Performance | Load tested at 2× peak, caching for catalogue | ✅ |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Pillar priority** | Derive from business context (regulated bank: security/reliability; startup MVP: cost/velocity). |
| **Depth** | Apply WAF proportionally to workload criticality. |
| **Continuous** | Re-assess after major changes, incidents and quarterly for critical workloads. |
| **Ownership** | Workload team owns WAF alignment; platform team provides landing zone guardrails. |
| **AI workloads** | Use WAF AI workload guidance in addition to pillars (Topic 34 file 01). |

## Hands-on

1. Read the design principles for each pillar; summarise each in one sentence for your organisation.
2. Pick a workload and set targets per pillar.
3. Identify three trade-offs in its design and record them.
4. Read the WAF service guide for two services you use and list gaps.
5. Map Advisor recommendations for the workload to pillars.

## Security, cost and reliability notes

- **Security and reliability** are often non-negotiable floors; cost and performance optimise within them.
- **Cost of WAF alignment** must be justified against risk and business value.
- **Over-engineering** is itself a reliability and cost risk.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating WAF as a one-time checklist | Continuous practice |
| Optimising one pillar in isolation | Explicit trade-offs |
| Ignoring service guides | Use per-service recommendations |
| Same rigour for every workload | Proportional to criticality |
| Confusing WAF with CAF | Workload vs adoption/platform |

## Check yourself

1. Name the five WAF pillars and their concerns.
2. What does each pillar provide?
3. Give three trade-offs between pillars.
4. How do WAF and CAF differ?
5. What are WAF service guides?
6. How should pillar priorities be decided?

<details>
<summary>Answers</summary>

1. Reliability (resiliency, availability, recovery), Security (CIA, threat protection), Cost Optimisation (cost modelling, waste), Operational Excellence (observability, DevOps, safe deployment), Performance Efficiency (scaling, capacity, performance).
2. Design principles, design review checklists, recommendations and trade-off guidance.
3. Any three: multi-region (reliability vs cost/complexity), private networking (security vs cost/friction), autoscale-in (cost vs performance), telemetry (operations vs cost), caching (performance vs consistency), reservations (cost vs flexibility).
4. WAF focuses on workload design quality; CAF on organisational cloud adoption and platform landing zones.
5. Per-service recommendations mapped to the five pillars.
6. From business context, criticality and risk appetite, with explicit agreement from stakeholders.

</details>

## References

- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [WAF pillars](https://learn.microsoft.com/azure/well-architected/pillars)
- [Service guides](https://learn.microsoft.com/azure/well-architected/service-guides/)
- [Workload guidance](https://learn.microsoft.com/azure/well-architected/workloads)
