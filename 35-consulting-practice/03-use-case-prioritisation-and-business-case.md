# Use-Case Prioritisation and Business Case

## Why it matters

After envisioning, most organisations have 30 ideas and budget for three. Choosing badly wastes a year: either picking glamorous but infeasible ideas, or easy ones that don't matter. A structured **prioritisation** (value vs feasibility, with risk and readiness) gets agreement on where to start, and a credible **business case** (benefits, costs, ROI, TCO, risks, success metrics) secures funding and sets the bar for success. Microsoft's Cloud Adoption Framework provides scoring guidance for AI agent use cases; consultants combine it with financial modelling ([Topic 29](../29-cost-management-and-finops/README.md)) and value measurement ([Topic 22 file 10](../22-evaluation-observability-and-genaiops/10-measuring-business-value.md)).

## Key concepts

### Scoring dimensions (CAF-aligned)

| Dimension | Criteria (score 1–5) |
|---|---|
| **Business impact** | Strategic alignment, business value (cost, speed, decisions, experience), change management timeframe |
| **Technical feasibility** | Implementation and operation risks known and mitigable, sufficient safeguards (compliance, security, responsible AI), technology fit with existing systems and data |
| **User desirability** | Clear value proposition for users, users want it, adoption likelihood |
| **Data readiness** | Data exists, accessible, quality, permissions, freshness |

Plot on a **value vs feasibility** matrix: quick wins (high/high), strategic bets (high value, lower feasibility), fill-ins (low value, high feasibility), avoid (low/low). Apply gates: no use case advances with unclear safeguards.

### Business case components

| Component | Content |
|---|---|
| **Problem and opportunity** | Current pain, volumes, costs, baseline metrics |
| **Solution summary** | Approach (buy/low-code/pro-code), scope, assumptions |
| **Benefits** | Quantified (hours saved × loaded cost, cost avoided, revenue uplift, error reduction) and qualitative (experience, risk) |
| **Costs** | One-off (build, migration, change management) and run (Azure/licences, operations, evaluation, support) |
| **Financials** | NPV, payback period, ROI, TCO over 3 years; scenarios (conservative/expected/optimistic) |
| **Risks** | Delivery, adoption, technical, regulatory, AI-specific (quality, safety) with mitigations |
| **Success metrics** | KPIs with baselines and targets; decision gates |
| **Plan** | Phases, milestones, resources, governance |

### Benefit realism

- Separate **capacity freed** from **cash saved** (hours saved only become savings if redeployed or avoided hiring).
- Apply **adoption curves** (not 100% from day one).
- Include **escalation and rework** costs for agents.
- Use **ranges** and sensitivity on key drivers.

### Decision gates

Pilot exit criteria and production go criteria use the business case metrics; CAF advises treating gates seriously and stopping or pivoting when results fall short.

## How it works on Azure

Scoring sheet example:

| Use case | Impact | Feasibility | Desirability | Data | Weighted score | Quadrant |
|---|---|---|---|---|---|---|
| Invoice processing automation | 4 | 4 | 4 | 4 | 4.0 | Quick win |
| Claims triage agent | 5 | 3 | 4 | 3 | 3.9 | Strategic bet |
| Marketing copy generator | 2 | 5 | 3 | 5 | 3.4 | Fill-in |
| Autonomous procurement negotiation | 4 | 1 | 2 | 2 | 2.4 | Avoid (for now) |

Business case snippet (invoice automation):

```text
Baseline: 12,000 invoices/month, 9 min each manual → 1,800 hours/month; error rate 3%
Expected: 70% straight-through, 2 min review for the rest → ~350 hours/month (−80%)
Value: 1,450 hours/month × €45 loaded cost × 12 = €783k/year capacity (cash only if redeployed/avoided hires)
Costs: build €180k one-off; run €6k/month Azure (extraction, models, Functions, monitoring) + €3k/month support
Payback: ~4–6 months (expected scenario); conservative (50% STP) ~9 months
KPIs: STP rate, field accuracy, cycle time, cost per invoice; gate: STP ≥ 50% in pilot
```

Azure inputs: pricing calculator and Retail Prices API for run costs, Azure Migrate business case and TCO calculator for migration-linked cases, Cost Management for validating after launch.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Weights** | Agree weighting with the sponsor before scoring. |
| **Portfolio balance** | Mix quick wins (momentum) with one or two strategic bets. |
| **Precision** | Business case precision proportional to investment size. |
| **Buy vs build** | Include buy options in cost comparisons (Topic 24 file 05). |
| **Ownership** | Each use case has a business owner accountable for benefits. |

## Hands-on

1. Score ten use cases with the four dimensions; plot the matrix.
2. Build a three-scenario business case for the top use case.
3. Identify the three most sensitive assumptions and test ranges.
4. Define pilot exit and production go criteria.
5. Present the business case to a peer acting as CFO; refine based on challenges.

## Security, cost and reliability notes

- **Include risk and compliance costs** (evaluation, red teaming, governance, security controls).
- **Don't hide run costs**: AI consumption costs scale with usage.
- **Reliability requirements** affect cost (Topic 28); include them.

## Common mistakes

| Mistake | Fix |
|---|---|
| Prioritising by excitement | Scoring with agreed criteria |
| Treating hours saved as cash | Capacity vs cash distinction |
| Single-point estimates | Scenarios and sensitivity |
| Ignoring adoption | Adoption curves and change costs |
| No gates | Decision criteria at each stage |

## Check yourself

1. What dimensions does CAF use to score AI use cases?
2. Describe the value vs feasibility quadrants.
3. What are the components of a business case?
4. Why distinguish capacity freed from cash saved?
5. Which financial measures should a business case include?
6. What makes benefits realistic?

<details>
<summary>Answers</summary>

1. Business impact, technical feasibility, user desirability (plus data readiness in practice).
2. Quick wins (high value, high feasibility), strategic bets (high value, lower feasibility), fill-ins (low value, high feasibility), avoid (low/low).
3. Problem/opportunity, solution, benefits, costs, financials, risks, success metrics, plan.
4. Freed hours only become financial savings if redeployed to value or used to avoid costs.
5. NPV, payback, ROI and multi-year TCO with scenarios.
6. Baselines, adoption curves, escalation/rework costs, ranges and sensitivity.

</details>

## References

- [Business plan for AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [AI strategy (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/strategy)
- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)
- [Azure TCO calculator](https://azure.microsoft.com/pricing/tco/calculator/)
