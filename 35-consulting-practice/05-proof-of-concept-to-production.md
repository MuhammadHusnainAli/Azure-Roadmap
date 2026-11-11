# Proof of Concept to Production

## Why it matters

"Pilot purgatory" is the most common fate of enterprise AI: impressive demos that never reach production because they were built on sample data, ignored security and governance, had no evaluation, no owner, no budget for run costs, or no agreed definition of success. A good consultant scopes proofs of concept (PoCs) and minimum viable products (MVPs) to answer specific questions, defines exit criteria upfront, builds with production in mind from day one, and plans the path to production before the first line of code. Microsoft's CAF recommends rapid pilots on the hardest steps and treating business metrics as decision gates.

## Key concepts

### PoC vs pilot vs MVP

| Stage | Question answered | Scope | Users | Typical duration |
|---|---|---|---|---|
| **PoC** | Is it technically feasible? Can AI do this task well enough? | Narrow, hardest steps, representative data | Team only | 2–4 weeks |
| **Pilot** | Does it create value with real users in a real process? | End-to-end for a subset | Limited real users | 4–12 weeks |
| **MVP** | What's the smallest production-grade product that delivers value? | Production quality, limited features | Real users at scale | Months |

### Exit criteria (define before starting)

| Category | Example criteria |
|---|---|
| **Quality** | Task completion ≥ 85% on a 200-case evaluation set; groundedness ≥ 4/5; no high-severity safety failures |
| **Value** | Handling time −30% in pilot group; user satisfaction ≥ 4/5 |
| **Feasibility** | Data access and permissions confirmed; integration with system X works |
| **Cost** | Cost per transaction ≤ €0.20 at projected volume |
| **Risk** | Security review passed; impact assessment approved; red teaming findings resolved |
| **Operability** | Monitoring, owner and support model identified |

Outcomes: **go** (proceed), **pivot** (change approach/scope), **stop** (document learnings).

### Avoiding pilot purgatory

| Cause | Prevention |
|---|---|
| No business owner | Named owner accountable for value |
| Demo data | Representative, governed real data from the start |
| Security/legal involved late | Involve them at kick-off; use landing zone and platform |
| No evaluation | Evaluation set and metrics from week one (Topic 22) |
| No production budget | Run-cost estimate and funding decision at pilot exit |
| Throwaway code | Production-minded architecture: IaC, CI/CD, identity, private networking where needed |
| Scope creep | Fixed questions, timebox, change control |
| No adoption plan | Change management planned during pilot (file 09) |

### Production readiness checklist

Architecture review (WAF/AI), security review and threat model, responsible AI sign-off, evaluation gates in CI, monitoring and alerts, runbooks and on-call, cost controls and budgets, DR plan, data protection and retention, support model and SLAs, training and communications, success metrics dashboard.

## How it works on Azure

Production-minded PoC setup on Azure:

```text
Sandbox subscription in the landing zone (policies applied, budget alerts)
Foundry project + AI gateway (shared platform if available)
Agent defined as code in Git; IaC for resources (Bicep/azd templates)
Evaluation dataset (50–200 cases) + Foundry evaluators in a pipeline
Tracing to Application Insights from day one
Representative data via governed access (no copies to laptops)
```

Pilot-to-production path:

```text
PoC (weeks 1–4): feasibility on hardest steps → exit review
Pilot (weeks 5–12): real users, end-to-end, value metrics, security/RAI reviews → go/no-go with business case
Production hardening (weeks 13–20): private networking, HA, DR, CI gates, monitoring, support → production go gate
Rollout: phased (rings), adoption programme, value tracking
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **What to prove first** | The riskiest assumption (often data quality, extraction accuracy or integration), not the UI. |
| **Buy vs build in the PoC** | Test buy options (Copilot, Copilot Studio) before building custom. |
| **Production-minded vs speed** | Use templates and platform to get both; avoid shortcuts that block production (keys, public endpoints, manual setup). |
| **Pilot size** | Large enough for statistically meaningful value metrics; small enough to manage risk. |
| **Stop decisions** | Celebrate informed stops; they save money. |

## Hands-on

1. Write a PoC charter: questions, scope, data, success/exit criteria, timebox, team.
2. Set up a production-minded PoC environment with IaC, tracing and an evaluation pipeline.
3. Define pilot value metrics with baselines.
4. Run a go/no-go review with the exit criteria table.
5. Draft the production readiness checklist for the use case.

## Security, cost and reliability notes

- **Real data in PoCs** must follow data protection rules; use governed access and the landing zone.
- **PoC cost** is small; production run cost can be large; estimate it during the PoC.
- **Reliability** expectations rise sharply at production; plan hardening time.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting without exit criteria | Define upfront |
| Proving the easy part | Prove the hardest steps |
| Building throwaway prototypes | Production-minded foundations |
| Security review at the end | Involve from the start |
| No decision at the end | Formal go/pivot/stop review |

## Check yourself

1. Distinguish PoC, pilot and MVP.
2. Give exit criteria for quality, value, cost and risk.
3. List five causes of pilot purgatory and their prevention.
4. What does a production-minded PoC environment include?
5. What's on a production readiness checklist?
6. Why should PoCs test the hardest steps first?

<details>
<summary>Answers</summary>

1. PoC: technical feasibility on narrow scope; pilot: value with real users in a real process; MVP: smallest production-grade product.
2. Quality: evaluation thresholds; value: time savings and satisfaction; cost: cost per transaction; risk: security review, impact assessment, red teaming resolved.
3. Any five: no owner (named owner), demo data (real governed data), late security/legal (involve early), no evaluation (datasets from week one), no production budget (run-cost estimate), throwaway code (IaC/CI), scope creep (timebox), no adoption plan (change management).
4. Sandbox subscription with policies and budgets, Foundry project and gateway, agents and resources as code, evaluation pipeline, tracing, governed data access.
5. Architecture and security reviews, RAI sign-off, evaluation gates, monitoring, runbooks, cost controls, DR, data protection, support model, training, success dashboard.
6. They carry the most risk; if they work, the rest is lower risk (CAF guidance).

</details>

## References

- [Business plan for AI agents: rapid piloting (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [GenAIOps for organisations with MLOps investments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/genaiops-for-mlops)
- [Evaluation in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/evaluation-overview)
- [Well-Architected AI workloads](https://learn.microsoft.com/azure/well-architected/ai/)
