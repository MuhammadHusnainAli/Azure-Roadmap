# Proposals, Estimation and Statements of Work

## Why it matters

Consulting engagements succeed or fail on scope: vague scope leads to disputes, overruns and unhappy clients; over-precise scope written too early locks in the wrong solution. A good **proposal** wins the right work by showing understanding of the client's problem and a credible approach; a good **estimate** is transparent about assumptions and uncertainty; a good **statement of work (SoW)** defines deliverables, responsibilities, acceptance and change control so both sides know what "done" means. AI projects add uncertainty (data quality, model performance) that must be handled with phased scope and explicit decision points.

## Key concepts

### Proposal structure

| Section | Content |
|---|---|
| **Executive summary** | Client situation, desired outcomes, recommended approach, investment, timeline |
| **Understanding** | Business context and challenges in the client's language |
| **Approach** | Phases, activities, methods (CAF/WAF, evaluation-driven AI), deliverables |
| **Solution overview** | High-level architecture and options considered (buy vs build) |
| **Team and roles** | Consultant team, client roles required, Microsoft involvement |
| **Timeline** | Phases and milestones with decision gates |
| **Investment** | Fees, Azure/licensing estimates (separately), assumptions |
| **Risks and mitigations** | Including AI-specific risks |
| **Why us** | Relevant experience, accelerators, partner designations |
| **Next steps** | Decision and start date |

### Effort estimation techniques

| Technique | Use |
|---|---|
| **Analogous (top-down)** | Early, based on similar past projects |
| **Bottom-up (WBS)** | Detailed work breakdown with task estimates |
| **Three-point (PERT)** | (Optimistic + 4 × Most likely + Pessimistic) ÷ 6 per task for uncertainty |
| **Parametric** | Units × rate (for example, per integration, per document type, per agent tool) |
| **Timeboxing** | Fixed duration for exploratory phases (PoC) |

Add explicit contingency (for example 10–25% depending on uncertainty) rather than hiding padding. Separate **services effort** from **Azure consumption and licensing** costs (Topic 29 file 08).

### AI-specific scoping

| Uncertainty | Scoping approach |
|---|---|
| Model quality on client data | Discovery/PoC phase with evaluation-based exit criteria before build commitment |
| Data quality and access | Data readiness tasks with client responsibilities and dependencies |
| Number of document types/intents | Parametric pricing per type; scope cap with change requests |
| Integration complexity | Spike/technical investigation tasks |
| Responsible AI and security reviews | Planned time and client approvals as dependencies |

### Statement of work

| Element | Content |
|---|---|
| **Objectives and scope** | In-scope and **out-of-scope** items explicitly |
| **Deliverables** | Named, with descriptions and formats |
| **Acceptance criteria** | Objective criteria and acceptance process per deliverable |
| **Responsibilities (RACI)** | Consultant, client, Microsoft, third parties |
| **Assumptions and dependencies** | Data access, environments, SMEs availability, licences, approvals |
| **Timeline and milestones** | Including decision gates |
| **Commercials** | Fixed price, time and materials (T&M), capped T&M, milestone payments |
| **Change control** | How changes are requested, assessed and approved |
| **Governance** | Steering committee, status reporting, escalation (file 11) |
| **Risks** | Shared risk register |

### Commercial models

| Model | Fit |
|---|---|
| **Fixed price** | Well-defined scope, low uncertainty |
| **Time and materials** | Exploratory or evolving scope |
| **Capped T&M** | Balance of flexibility and budget certainty |
| **Phased fixed price** | Discovery/PoC fixed, then build priced after decision gate |
| **Outcome-based** | Shared risk on business outcomes (requires mature measurement) |

## How it works on Azure

Estimate breakdown example (RAG assistant MVP):

| Workstream | Optimistic | Most likely | Pessimistic | PERT (days) |
|---|---|---|---|---|
| Landing zone integration and networking | 5 | 8 | 15 | 8.7 |
| Ingestion pipeline (3 sources) | 8 | 12 | 20 | 12.7 |
| Agent and tools | 6 | 10 | 16 | 10.3 |
| Evaluation dataset and CI gates | 4 | 6 | 10 | 6.3 |
| Security, RAI reviews, red teaming | 3 | 5 | 8 | 5.2 |
| UI/Teams integration | 4 | 6 | 10 | 6.3 |
| Monitoring, runbooks, handover | 3 | 5 | 8 | 5.2 |
| **Total** | | | | **54.7 + 15% contingency ≈ 63 days** |

Azure run-cost estimate attached separately from the pricing calculator with assumptions (Topic 29 file 08).

Use Azure DevOps or GitHub Projects to turn the WBS into a backlog after signature.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Phasing** | Separate discovery/PoC from build for AI work with uncertain feasibility. |
| **Assumption transparency** | List assumptions explicitly; they protect both parties. |
| **Client responsibilities** | Name SMEs, data access and approvals with dates; delays become change requests. |
| **Acceptance** | Objective criteria (evaluation thresholds for AI) avoid subjective disputes. |
| **Pricing model** | Match to uncertainty. |

## Hands-on

1. Write a two-page proposal for an AI assistant engagement.
2. Build a WBS and PERT estimate with contingency.
3. Draft an SoW with in/out of scope, deliverables, acceptance criteria, RACI and assumptions.
4. Define a change control process.
5. Peer-review another consultant's estimate and challenge assumptions.

## Security, cost and reliability notes

- **Security and compliance work** must be in scope, not assumed.
- **Azure consumption** during delivery (dev/test environments) needs an owner and budget.
- **Warranties and support** post go-live must be explicit.

## Common mistakes

| Mistake | Fix |
|---|---|
| Fixed price for uncertain AI feasibility | Phased commercials with gates |
| No out-of-scope list | Explicit exclusions |
| Subjective acceptance | Objective criteria |
| Mixing fees and Azure costs | Separate estimates |
| Ignoring client dependencies | Responsibilities with dates |

## Check yourself

1. What sections should a proposal include?
2. Describe five estimation techniques.
3. How do you scope AI uncertainty?
4. What must an SoW contain?
5. Compare commercial models.
6. How does PERT estimation work?

<details>
<summary>Answers</summary>

1. Executive summary, understanding, approach, solution overview, team, timeline, investment, risks, why us, next steps.
2. Analogous, bottom-up WBS, three-point/PERT, parametric, timeboxing.
3. Discovery/PoC phases with evaluation exit criteria, data readiness tasks, parametric pricing with caps, technical spikes, planned review time.
4. Objectives, in/out of scope, deliverables, acceptance criteria, RACI, assumptions/dependencies, timeline, commercials, change control, governance, risks.
5. Fixed price (defined scope), T&M (exploratory), capped T&M (balanced), phased fixed price (gated), outcome-based (shared risk).
6. (Optimistic + 4 × most likely + pessimistic) ÷ 6 per task, summed, plus explicit contingency.

</details>

## References

- [CAF: plan your migration and adoption](https://learn.microsoft.com/azure/cloud-adoption-framework/plan/)
- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)
- [Azure Boards agile project management](https://learn.microsoft.com/azure/devops/boards/best-practices-agile-project-management)
- [Well-Architected cost optimisation](https://learn.microsoft.com/azure/well-architected/cost-optimization/)
