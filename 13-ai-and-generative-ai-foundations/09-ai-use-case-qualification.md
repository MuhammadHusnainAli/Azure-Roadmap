# AI Use Case Qualification

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ○ A ● C ● · Status checked: 2026-11

## Why it matters

The most expensive AI mistake is building the wrong thing. Many proofs of concept never reach production because the use case had no clear value, the data wasn't ready, the risk was unacceptable, or a simpler non-AI solution would have worked. **Qualification** is the structured check, done before building, of whether a problem is a good AI problem: is it feasible, valuable, safe and supported by data? Consultants run it in discovery workshops; architects use it to scope; engineers use it to push back on unrealistic requirements.

## Key concepts

### Is it an AI problem at all?

| Signal it suits generative AI | Signal it doesn't |
|---|---|
| Work involves unstructured language, documents, images or audio | Work is fully rules-based and deterministic (use code or workflow) |
| Many variations of inputs make hand-written rules impractical | A lookup table or SQL query answers it |
| Some errors are acceptable or catchable by a human or validation | Every error is unacceptable and can't be detected |
| Output is a draft, suggestion, classification or extraction that can be checked | Output is a legally binding decision with no review |
| Volume or delay makes manual handling costly | Volume is tiny, so manual is cheaper |
| For predictions from tabular history: classic ML may be better | |

### The four lenses

| Lens | Key questions |
|---|---|
| **Value** | What business metric improves (time, cost, revenue, quality, risk)? By how much? Who benefits, and how often? What happens today? |
| **Feasibility** | Can current models do this task at the needed quality? Is there a reference solution or pattern? What integrations are needed? Latency and volume requirements? Skills available? |
| **Data readiness** | Does the needed data exist, is it accessible, current, high quality, permissioned and legally usable for this purpose? (topic 12 file 12) |
| **Risk and responsibility** | What harm could errors cause, and to whom? Are there regulatory constraints (EU AI Act risk category, sector rules, privacy)? Is human oversight possible? Fairness and transparency obligations? |

Add **adoption** for consultants: will users trust and use it? Does it fit their workflow? Who owns it after go-live?

### A scoring template

Score each 1 (poor) to 5 (strong):

| Criterion | Weight | Example: "HR policy assistant" | Example: "Automated loan approval" |
|---|---|---|---|
| Business value | 25% | 3: saves HR time, moderate volume | 5: high volume, revenue impact |
| Technical feasibility | 20% | 5: standard RAG pattern | 3: possible but needs explainability |
| Data readiness | 20% | 3: policies scattered, some outdated | 4: structured history exists |
| Risk (inverse: 5 = low risk) | 20% | 4: low harm, answers cite sources | 1: high-risk decision, regulatory scrutiny |
| Adoption and ownership | 15% | 4: HR champion, Teams-based | 2: process change, union concerns |
| **Weighted score** | | **3.8 → proceed** | **3.1 → redesign as decision support with human approval, or classic ML with explainability** |

Use the score to rank a backlog and start conversations, not as an automatic decision.

### Value estimation

```text
Annual value ≈ volume × (time saved per item × loaded cost per hour + error-cost reduction) × adoption rate
Annual cost  ≈ model/token cost + platform (search, hosting, gateway) + build + run/support + change management
```

Be honest about the **adoption rate** and the **human review time** that remains. A 70% time saving with 40% adoption is a 28% saving.

### Feasibility checks

| Check | How |
|---|---|
| Can a model do it? | 1–2 day prototype in the Foundry playground with 20–50 real examples |
| Quality bar | Agree acceptable accuracy, groundedness and error types with the business up front |
| Latency and scale | Requests per minute, peak load, response time needs → model choice, quotas, PTU (topic 15) |
| Integration | Systems to read from and write to, APIs, authentication, network constraints |
| Cost per transaction | Tokens per request × price, plus retrieval and hosting, against value per transaction |
| Operability | Who monitors quality, handles escalations and updates content? |

### Risk triage

| Risk level | Characteristics | Typical controls |
|---|---|---|
| **Low** | Internal, informational, reversible, human reads the output | Content filters, citations, feedback, basic evaluation |
| **Medium** | Customer-facing, influences decisions, uses personal data | Plus: red teaming, groundedness evaluation, human escalation, privacy review, monitoring |
| **High** | Affects rights, safety, finances or employment; regulated (for example EU AI Act high-risk categories) | Plus: impact assessment, human approval of decisions, explainability, audit trail, legal sign-off. Consider whether AI should decide at all |

Microsoft's **Responsible AI Standard** and **impact assessment** templates provide a structured approach (topic 23). Some uses (for example manipulative or social-scoring systems) are prohibited by regulation.

## How it works on Azure

| Qualification activity | Azure or Microsoft tooling |
|---|---|
| Rapid feasibility prototype | Foundry portal playgrounds, model catalogue and benchmarks, agent playground |
| Compare models on your examples | Foundry evaluations and model leaderboards (topic 15) |
| Estimate cost | Azure pricing calculator, token counts from prototypes, AI gateway token metrics |
| Data readiness scan | Microsoft Purview (classification, quality), Fabric OneLake catalogue |
| Responsible AI assessment | Microsoft Responsible AI Impact Assessment template, Foundry safety evaluations and AI red teaming agent |
| Adoption planning | Cloud Adoption Framework for AI (strategy, plan, ready, govern, manage, secure) |

### Qualification workshop agenda (half day)

```text
1. Problem framing (45 min): current process, pain, metrics, users
2. Use case ideation and pattern mapping (45 min): which pattern (file 07)?
3. Scoring (60 min): value, feasibility, data, risk, adoption
4. Data walk-through (30 min): sources, owners, access, quality
5. Risk triage (30 min): harms, regulation, human oversight
6. Decision and next step (30 min): prototype scope, success criteria, owners
```

Outputs: ranked backlog, one-page use case canvas per candidate, success criteria, and a 2–4 week proof-of-value plan.

### Use case canvas (one page)

```text
Problem & users · Current process & baseline metrics · Proposed AI pattern
Inputs & data sources (owner, access, quality) · Outputs & where they go
Success metrics & quality bar · Human-in-the-loop design
Risks & controls · Cost estimate per transaction & per year · Value estimate
Dependencies & integrations · Owner after go-live · Decision: go / redesign / stop
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Proof of concept vs proof of value | Prefer proof of value: real data, real users, measured against a baseline, with production constraints (security, cost) considered from day one |
| Quick wins vs strategic bets | Balance the portfolio: a few low-risk internal wins to build skills and trust, plus one or two strategic use cases with clear sponsors |
| Stop criteria | Define in advance what result means stop or pivot (for example below 80% accuracy after two iterations) |
| Buy vs build | Check whether Microsoft 365 Copilot, Dynamics 365 or an ISV product already solves it before building (file 08) |

## Hands-on

1. Take three candidate use cases from your organisation (or invent them for a retailer, an insurer and a manufacturer). Fill a use case canvas for each.
2. Score them with the weighted template and rank them.
3. For the top one, build a one-day prototype in the Foundry playground with 30 real examples and record the quality you get.
4. Estimate monthly token cost from the prototype's usage and compare it with the value estimate.
5. Write the stop/go criteria for a 4-week proof of value.

## Security, cost and reliability notes

- **Security:** during qualification, flag any data that can't legally or contractually be processed by AI services or in certain regions.
- **Cost:** include run costs (tokens, search, hosting, monitoring, people), not just build costs. Cost per transaction must be below value per transaction.
- **Reliability:** quality targets must be measurable and owned; without an evaluation set there's no way to say the solution works.

## Common mistakes

- Starting from technology ("we need an agent") instead of the problem.
- No baseline metrics, so value can't be proven.
- Ignoring data readiness until the build phase.
- Underestimating change management and human review time.
- Treating high-risk decisions as fully automatable.
- Endless proofs of concept with no stop criteria or production path.

## Check yourself

1. What signals suggest a problem isn't a good fit for generative AI?
2. Name the four qualification lenses (plus the extra one for consultants).
3. How do you estimate the annual value of a use case?
4. What makes a use case high risk, and what controls apply?
5. What's the difference between a proof of concept and a proof of value?
6. What should a use case canvas include?

<details>
<summary>Answers</summary>

1. Fully deterministic rules, simple lookups, zero error tolerance with no way to detect errors, legally binding decisions without review, very low volume, or tabular prediction better served by classic ML.
2. Value, feasibility, data readiness, risk and responsibility (plus adoption and ownership).
3. Volume × (time saved × cost per hour + error-cost reduction) × realistic adoption rate, compared with total annual cost.
4. Effects on people's rights, safety, finances or employment, or regulated categories. Controls include impact assessments, human approval, explainability, audit trails, red teaming, monitoring and legal sign-off.
5. A proof of concept shows something is technically possible; a proof of value measures business impact with real data and users against a baseline and production constraints.
6. Problem and users, baseline, AI pattern, data sources, outputs, success metrics, human-in-the-loop design, risks and controls, cost and value estimates, dependencies, owner, and the decision.

</details>

## References

- [AI adoption with the Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/)
- [AI strategy: identify use cases (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/strategy)
- [Microsoft Responsible AI Standard and impact assessment](https://www.microsoft.com/ai/principles-and-approach)
- [Responsible AI in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/responsible-use-of-ai-overview)
- [Evaluate generative AI apps in Foundry](https://learn.microsoft.com/azure/foundry/concepts/evaluation-approach-gen-ai)
- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)
