# The Azure Consultant Role

## Why it matters

Organisations rarely fail at cloud and AI for lack of technology; they fail because priorities are unclear, business cases are weak, risks are surprises, teams aren't ready and projects drift from pilot to nowhere. **Azure consultants** close that gap: they connect business outcomes to technical designs, guide decisions, structure delivery and leave clients more capable. The role spans advisory (strategy, assessments), design (architecture, estimates) and delivery leadership, often alongside Microsoft account teams and partners. This file defines the role, engagement types, responsibilities and how consultants work within the Microsoft ecosystem.

## Key concepts

### Consultant vs architect vs engineer

| Role | Primary focus | Typical outputs |
|---|---|---|
| **AI/cloud engineer** | Building and operating solutions | Code, IaC, pipelines, working systems |
| **Solution architect** | Designing solutions that meet requirements | Architecture, ADRs, designs, reviews |
| **Consultant** | Business outcomes, decisions, change | Assessments, business cases, roadmaps, proposals, executive communication, engagement management |

Good consultants are technically credible (they can read an architecture and challenge an estimate) and business-fluent (they can explain value, risk and trade-offs to executives).

### Engagement types

| Engagement | Purpose | Typical duration | Key deliverables |
|---|---|---|---|
| **Envisioning / discovery workshop** | Identify opportunities, align stakeholders | 1–3 days | Use-case backlog, priorities, next steps (file 02) |
| **Assessment** | Current state, readiness, gaps | 2–6 weeks | AI readiness report (file 04), Well-Architected Review, migration assessment |
| **Strategy and roadmap** | Direction, sequencing, business case | 4–8 weeks | Strategy, business case, roadmap (file 03) |
| **Proof of concept / pilot** | Validate feasibility and value | 2–8 weeks | Working prototype, evaluation results, go/no-go (file 05) |
| **Design** | Architecture and plan for build | 2–6 weeks | HLD/LLD, estimates, SoW for build (files 06–07) |
| **Delivery / implementation leadership** | Build and deploy | Months | Working solution, governance, handover (file 11) |
| **Managed service / advisory retainer** | Ongoing optimisation and support | Ongoing | Reviews, FinOps, roadmap updates |
| **Adoption and change** | Users actually use the solution | Weeks–months | Adoption plan, training, metrics (file 09) |

### Responsibilities

| Area | Responsibilities |
|---|---|
| **Client outcomes** | Define success, measure value, keep focus on outcomes over outputs |
| **Trusted advice** | Recommend what's right (including "don't build this"), disclose trade-offs and risks |
| **Engagement management** | Scope, plan, risks, budget, stakeholder management, status reporting |
| **Quality** | Peer review of designs and estimates, Microsoft guidance alignment |
| **Knowledge transfer** | Leave the client able to operate and evolve |
| **Commercial** | Proposals, change requests, account growth (ethically) |
| **Responsible AI** | Raise ethical, legal and safety considerations early |

### Working with Microsoft and partners

| Party | Role |
|---|---|
| **Microsoft account team** | Account executives, specialists, solution engineers, customer success; can bring programmes, funding and product expertise |
| **Microsoft partners** | System integrators, ISVs, managed service providers; deliver services and solutions; Microsoft AI Cloud Partner Program designations and specialisations |
| **Microsoft programmes** | Investment and deployment-assistance programmes (for example Azure investment programmes such as Frontier Accelerate for Azure with Cloud Accelerate Factory at the time of writing; names and eligibility change frequently) |
| **Microsoft Learn and Architecture Center** | Official guidance to cite in deliverables |

## How it works on Azure

Consultant toolkit mapped to Azure resources:

| Need | Resource |
|---|---|
| Strategy and adoption | Cloud Adoption Framework (strategy, plan, ready, AI and agents scenarios) |
| Architecture quality | Well-Architected Framework and assessments |
| Reference designs | Azure Architecture Center |
| Cost and value | Pricing calculator, TCO calculator, Azure Migrate business case, Cost Management |
| Readiness | Microsoft Learn Assessments (cloud adoption, AI readiness, WAF) |
| Migration | Azure Migrate |
| Governance | Azure Policy, landing zones, Compliance Manager |
| Adoption | Microsoft Adoption hub, Copilot enablement resources |

A typical first 30 days with a new client: stakeholder interviews → current-state review (Azure estate via Resource Graph, Advisor, Defender, Cost) → envisioning workshop → prioritised use cases → proposal for assessment or pilot.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Advisory vs delivery** | Clarify which hat you're wearing; avoid recommending work only to sell it. |
| **Breadth vs depth** | Broad assessments first; deep dives where value or risk is highest. |
| **Microsoft alignment** | Use Microsoft frameworks for credibility, adapted to client context. |
| **Independence** | Recommend non-Microsoft or "do nothing" options when they're right. |
| **Scope control** | Explicit assumptions and change control to protect outcomes and margins. |

## Hands-on

1. Write a one-page description of your consulting offer (engagement types, outcomes, durations).
2. Map a client's needs to engagement types and sequence them.
3. Prepare a first-30-days plan for a new client.
4. Build a reusable toolkit folder (templates, Microsoft references, checklists).
5. Practise explaining one technical decision to a non-technical stakeholder in two minutes.

## Security, cost and reliability notes

- **Client data handling**: NDAs, least-privilege access to client environments, secure storage of deliverables.
- **Professional liability**: document assumptions and limitations of advice.
- **Ethics**: disclose conflicts of interest (partner incentives, licensing).

## Common mistakes

| Mistake | Fix |
|---|---|
| Leading with technology | Lead with outcomes |
| Saying yes to everything | Prioritise and challenge |
| No measurable success criteria | Define them upfront |
| Ignoring change management | Plan adoption from the start |
| Hiding risks to win work | Transparent risk communication |

## Check yourself

1. How does a consultant differ from an architect and an engineer?
2. List six engagement types and their deliverables.
3. What responsibilities does a consultant hold?
4. Who are the key Microsoft ecosystem parties?
5. Which Microsoft frameworks support consulting work?
6. What should the first 30 days with a client include?

<details>
<summary>Answers</summary>

1. Consultants focus on business outcomes, decisions and change; architects on solution design; engineers on building and operating.
2. Envisioning (use-case backlog), assessment (readiness report), strategy (roadmap, business case), PoC/pilot (prototype, go/no-go), design (HLD/LLD, estimate), delivery (solution, handover), managed service (reviews), adoption (adoption plan, metrics).
3. Client outcomes, trusted advice, engagement management, quality, knowledge transfer, commercial management, responsible AI.
4. Microsoft account teams, partners (SIs, ISVs, MSPs), Microsoft programmes, and Microsoft Learn/Architecture Center guidance.
5. CAF, WAF, Architecture Center, Learn Assessments, pricing/TCO tools, Azure Migrate.
6. Stakeholder interviews, current-state review, envisioning, prioritised use cases and a proposal for next steps.

</details>

## References

- [Cloud Adoption Framework strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/strategy/)
- [AI agents adoption (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/)
- [Microsoft Learn Assessments](https://learn.microsoft.com/assessments/)
- [Partner Center overview](https://learn.microsoft.com/partner-center/overview)
- [Azure investment programmes](https://azure.microsoft.com/solutions/azure-accelerate/)
