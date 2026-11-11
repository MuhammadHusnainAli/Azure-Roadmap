# Measuring Business Value

## Why it matters

An agent with a groundedness score of 4.6 and a 0% attack success rate can still be a failure if nobody uses it, it saves no time, or it costs more to run than the process it replaced. Executives fund AI for outcomes such as lower cost per case, faster cycle times, higher customer satisfaction and new revenue, not for evaluator scores. Microsoft's Cloud Adoption Framework (CAF) tells you to define success metrics and baselines **before** development and to use them as **go/no-go decision gates** at every stage. This file connects the technical telemetry from this topic to the business case, which is what architects and consultants are asked to defend.

## Key concepts

### Four value areas (CAF)

| Area | Impact | Example agent |
|---|---|---|
| **Reshape business processes** | Internal | Reads invoices, checks policy, updates ERP, completes multistep tasks |
| **Enrich employee experiences** | Internal | Gathers data from several sources and drafts a summary for planning |
| **Reinvent customer engagement** | Customer | Interprets open-ended questions and resolves them with personalised answers |
| **Accelerate innovation** | Customer | Scans market signals and highlights insights for product teams |

### Metric layers

| Layer | Question | Examples | Source |
|---|---|---|---|
| **Business outcome** | Did the KPI move? | Cost per case, cycle time, CSAT/NPS, revenue, error/rework rate, compliance findings | Business systems, finance |
| **Task effectiveness** | Did the agent do the job? | Task completion / resolution rate, escalation rate, containment, first-contact resolution | Evaluations, traces, Copilot Studio analytics |
| **Adoption** | Are people using it? | Active users, sessions per user, retention, % of eligible work routed through the agent | Telemetry, Copilot dashboard |
| **Quality and trust** | Is it good and safe enough? | Evaluator scores, thumbs up/down, ASR, incidents | Foundry evaluation and monitoring |
| **Cost** | What does it cost? | Tokens and model cost per task, platform cost, people cost to operate | Cost Management, gateway metrics |

Track a small number of KPIs per agent: one or two business outcomes, one task-effectiveness metric, one adoption metric, plus guardrail metrics for quality, safety and cost.

### Time saved, done properly

Self-reported time saved is optimistic. Better methods, in increasing rigour:

1. **Task-time studies**: measure the same task with and without the agent for a sample of users.
2. **Process telemetry**: compare cycle times in the system of record (ticket open-to-close, invoice receipt-to-post) before and after.
3. **Controlled rollout**: staggered deployment or A/B groups, comparing outcomes over the same period.

Then ask what happens to the saved time: avoided hiring, redeployment to higher-value work, faster revenue. Saved minutes that aren't redeployed don't show up in financials.

### Agent ROI

```text
Annual benefit  = (cases handled by agent × cost saved per case)
                + (revenue uplift attributable) + (risk/rework reduction)
Annual cost     = model and platform consumption + build (amortised) + run/operate people
                + evaluation, monitoring and red teaming + change management
ROI             = (benefit − cost) / cost
```

Cost saved per case must account for escalations: a case the agent starts but a human finishes still costs human time (and sometimes more, if the hand-off is poor).

### Decision gates

| Gate | Evidence |
|---|---|
| **Qualify** | Baseline KPI measured; target agreed; feasibility and safeguards assessed ([Topic 13 file 09](../13-ai-and-generative-ai-foundations/09-ai-use-case-qualification.md)) |
| **Pilot exit** | Task completion on the hardest steps; early user feedback; projected cost per task |
| **Production go** | Evaluation and safety gates passed; adoption plan; owner and support model |
| **Post-deployment review** (quarterly) | KPI vs target; adoption trend; cost trend; decide refine, scale, or retire |

## How it works on Azure

| Signal | Where |
|---|---|
| Task completion, intent resolution, rubric scores | Foundry continuous and scheduled evaluation (file 07) |
| Escalations, resolved/abandoned outcomes, CSAT | Copilot Studio analytics (Monitor page); custom spans/events for Foundry agents |
| User feedback | Foundry end-user feedback logged against traces |
| Tokens and cost per agent | API Management gateway token metrics, Azure Cost Management with tags per agent/team |
| Microsoft 365 Copilot adoption and impact | Copilot Dashboard in Viva Insights ([Topic 24](../24-copilot-and-low-code-agents/README.md)) |
| Business KPIs | Source systems (CRM, ITSM, ERP) joined in Fabric or Power BI |

Emit a business event span when a task completes, so you can join technical telemetry with outcomes:

```python
from opentelemetry import trace
tracer = trace.get_tracer("support-app")

with tracer.start_as_current_span("business.case_outcome") as span:
    span.set_attribute("case.id_hash", case_hash)          # pseudonymous
    span.set_attribute("case.outcome", "resolved")         # resolved | escalated | abandoned
    span.set_attribute("case.handle_seconds", handle_seconds)
    span.set_attribute("agent.version", "13")
```

```kusto
dependencies
| where name == "business.case_outcome" and timestamp > ago(30d)
| summarize cases=count(),
            resolved=countif(tostring(customDimensions["case.outcome"]) == "resolved"),
            escalated=countif(tostring(customDimensions["case.outcome"]) == "escalated")
  by agent_version = tostring(customDimensions["agent.version"])
| extend containment = todouble(resolved) / cases
```

Combine with cost per agent (tag resources and use gateway metrics) to report **cost per resolved case** by agent version.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **KPI selection** | Choose metrics the agent directly influences; avoid vanity metrics (messages sent). |
| **Baseline** | Measure before launch; for new processes, estimate and refine. |
| **Attribution** | Use controlled rollouts where possible; otherwise compare against trends and note confounders. |
| **Leading vs lagging** | Adoption and task completion lead; cost and CSAT lag. Report both. |
| **Portfolio view** | Compare agents on value and cost; retire low-value agents (agent sprawl costs money and risk). |
| **Qualitative evidence** | User interviews and feedback explain the numbers. |

## Hands-on

1. Pick an agent use case; write its value area, two business KPIs with baselines, one task metric, one adoption metric and guardrail metrics.
2. Instrument a `business.case_outcome` span; build the containment query by version.
3. Build a cost-per-resolved-case calculation using token costs from gateway metrics.
4. Draft an ROI model with conservative, expected and optimistic scenarios, including escalation costs.
5. Define the post-deployment review agenda and the criteria to scale, refine or retire.

## Security, cost and reliability notes

- **Privacy**: business telemetry must not leak personal data; use pseudonymous IDs and aggregate reporting.
- **Employee analytics**: productivity metrics about individuals raise legal and works-council concerns; report at team level with appropriate approvals.
- **Cost transparency**: include evaluation, monitoring and red teaming in run costs.
- **Gaming**: metrics like containment can be gamed (agent refuses to escalate); pair them with CSAT and quality guardrails.

## Common mistakes

| Mistake | Fix |
|---|---|
| No baseline | Measure current process first |
| Reporting evaluator scores as business value | Tie to KPIs and cost |
| Counting self-reported time saved as savings | Telemetry and controlled comparisons; track redeployment |
| Ignoring escalation and rework costs | Include human effort after hand-off |
| Never retiring agents | Quarterly portfolio review |

## Check yourself

1. Name CAF's four value areas for AI agents.
2. List the five metric layers and an example of each.
3. Why is self-reported time saved weak evidence, and what's better?
4. What goes into the cost side of agent ROI?
5. How should business metrics be used during development?
6. Why pair containment with CSAT?

<details>
<summary>Answers</summary>

1. Reshape business processes, enrich employee experiences, reinvent customer engagement, accelerate innovation.
2. Business outcome (cost per case), task effectiveness (resolution rate), adoption (active users), quality and trust (evaluator scores, ASR), cost (tokens per task).
3. It's optimistic and unverified; use task-time studies, process telemetry or controlled rollouts, and check whether saved time is redeployed.
4. Model/platform consumption, amortised build, operating people, evaluation/monitoring/red teaming, change management, and human effort on escalations.
5. As go/no-go decision gates at qualification, pilot exit, production go and post-deployment reviews.
6. Containment alone can be gamed by not escalating; CSAT and quality metrics reveal whether customers were actually helped.

</details>

## References

- [Business plan for AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [AI strategy (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/strategy)
- [Copilot Studio analytics overview](https://learn.microsoft.com/microsoft-copilot-studio/analytics-overview)
- [Microsoft 365 Copilot Dashboard (Viva Insights)](https://learn.microsoft.com/viva/insights/org-team-insights/copilot-dashboard)
- [Log end-user feedback in Foundry](https://learn.microsoft.com/azure/foundry/observability/how-to/log-end-user-feedback)
