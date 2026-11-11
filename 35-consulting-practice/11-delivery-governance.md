# Delivery Governance

## Why it matters

AI projects carry more uncertainty than typical IT projects: model quality on real data is unknown until tested, requirements evolve as users see what's possible, responsible AI and security reviews can change scope, and dependencies on data owners and other teams are common. Without governance, projects drift, overspend or ship something unsafe; with heavyweight governance, they never ship. **Delivery governance** for AI combines agile delivery, explicit risk management, quality gates (including evaluation and safety gates) and lightweight but effective decision forums. This file gives consultants and delivery leads a practical model.

## Key concepts

### Agile delivery for AI

| Practice | AI adaptation |
|---|---|
| **Sprints (2 weeks)** | Include evaluation runs and dataset growth as sprint work |
| **Backlog** | Features plus evaluation, data, safety and operability items |
| **Definition of ready** | Data access confirmed, acceptance criteria include evaluation metrics |
| **Definition of done** | Code reviewed, tests and **evaluation gates passed**, tracing in place, docs/ADRs updated, security checks passed |
| **Sprint reviews** | Demo plus metrics (quality, cost, latency) |
| **Spikes** | Timeboxed experiments for model/approach choices |
| **Dual-track** | Discovery (experiments, evaluation) alongside delivery |

### Governance structure

| Forum | Members | Cadence | Purpose |
|---|---|---|---|
| **Steering committee** | Sponsor, business owner, delivery lead, architect, client IT lead | Monthly / at gates | Direction, decisions, budget, escalations |
| **Project board / weekly status** | Delivery leads, product owner | Weekly | Progress, risks, dependencies |
| **Design authority / architecture review** | Architects, security, platform | As needed | Approve significant designs (ADRs) |
| **Responsible AI review** | RAI lead, legal, risk | At gates | Impact assessment, safety evidence |
| **Change control board** | Sponsor delegate, delivery lead | As needed | Scope changes |

### Quality gates

| Gate | Evidence |
|---|---|
| **Design gate** | HLD approved, threat model, data protection assessment, RAI impact assessment |
| **Build gates (CI)** | Tests, code scanning, IaC scanning, evaluation thresholds |
| **Pre-production gate** | Full evaluation, red teaming, performance/load tests, security review, DR test, runbooks |
| **Go-live gate** | Business readiness (training, comms), support model, monitoring and alerts, rollback plan |
| **Post-go-live review** | KPIs vs targets, incidents, adoption, lessons learned |

### Risk management

A **risk register** with ID, description, probability, impact, owner, mitigation, trigger, status. AI-specific risks: model quality below threshold, data access delays, prompt injection/safety findings, model retirement or capacity limits, cost overruns from token usage, adoption failure, regulatory changes. Review weekly; escalate high risks to steering.

### RAID log

Risks, Assumptions, Issues, Dependencies tracked together; dependencies (data owners, security approvals, platform changes) are frequent blockers in AI projects.

### Reporting

Weekly status (RAG), burn-up and budget, key metrics trend (evaluation scores, latency, cost per request), RAID summary, decisions needed.

## How it works on Azure

Tooling:

| Need | Tool |
|---|---|
| Backlog, sprints, boards | Azure Boards or GitHub Projects |
| Definition of done enforcement | Branch protection, required checks, evaluation gates (Topic 27) |
| Risk and RAID | Azure Boards work item types or a shared register |
| Metrics dashboards | Azure DevOps dashboards, Power BI, Application Insights workbooks |
| Evidence for gates | Foundry evaluation reports, red teaming scorecards, Secure Score, load test results, cost forecasts |

Example DoD for an agent feature:

```text
✓ PR reviewed (code owners), unit tests pass
✓ Evaluation: task adherence ≥ 0.85, groundedness ≥ 4.0, no high-severity safety failures vs baseline
✓ Tracing spans present for new tools; alerts updated
✓ Tool has least-privilege identity; approval required for write actions
✓ ADR updated if design changed; user-facing docs updated
✓ Cost per run within budget in staging
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Governance weight** | Proportional to risk tier and investment. |
| **Fixed scope vs adaptive** | Fixed outcomes and gates; adaptive scope within them. |
| **Gate evidence automation** | Automate evidence (CI, dashboards) to keep gates cheap. |
| **Decision speed** | Empower product owner for day-to-day decisions; escalate only material changes. |
| **Client involvement** | Joint governance with client roles named in the SoW. |

## Hands-on

1. Write a definition of ready and definition of done for an AI feature.
2. Set up a backlog with features, evaluation and safety work items.
3. Create a RAID log with ten AI-specific entries.
4. Define the gates and evidence for a project; automate two pieces of evidence.
5. Produce a one-page weekly status report.

## Security, cost and reliability notes

- **Security and RAI gates** must not be skipped under deadline pressure; escalate instead.
- **Cost tracking** includes Azure consumption in dev/test as well as fees.
- **Operational readiness** is a gate, not a post-launch activity.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating AI like deterministic software | Evaluation in DoD and gates |
| Risk register created and forgotten | Weekly review |
| Heavy governance slowing everything | Proportional, automated evidence |
| Ignoring dependencies | RAID log and escalation |
| Gates without objective criteria | Measurable thresholds |

## Check yourself

1. How does agile delivery adapt for AI?
2. What belongs in an AI definition of done?
3. Describe the governance forums.
4. List the quality gates and their evidence.
5. Give five AI-specific risks.
6. What is a RAID log?

<details>
<summary>Answers</summary>

1. Evaluation and dataset work in sprints, metrics in reviews, spikes for experiments, dual-track discovery and delivery, evaluation gates in DoD.
2. Reviewed code, tests, evaluation thresholds, tracing, least-privilege tools, docs/ADRs, cost within budget.
3. Steering committee, weekly project board, design authority, responsible AI review, change control board.
4. Design (HLD, threat model, RAI assessment), build (tests, scans, evaluation), pre-production (full evaluation, red teaming, load, security, DR), go-live (readiness, support, monitoring, rollback), post-go-live review.
5. Any five: model quality shortfall, data access delays, safety findings, model retirement/capacity, token cost overruns, adoption failure, regulatory change.
6. A log of Risks, Assumptions, Issues and Dependencies reviewed regularly.

</details>

## References

- [Azure Boards agile project management](https://learn.microsoft.com/azure/devops/boards/best-practices-agile-project-management)
- [Run evaluations in CI/CD](https://learn.microsoft.com/azure/foundry/how-to/evaluation-github-action)
- [Govern AI (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/govern)
- [Well-Architected operational excellence](https://learn.microsoft.com/azure/well-architected/operational-excellence/principles)
