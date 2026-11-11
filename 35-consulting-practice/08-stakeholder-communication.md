# Stakeholder Communication

## Why it matters

Good architecture rarely wins on its own merits; it wins when the right people understand it, trust it and act on it. Executives decide in minutes based on a story about value and risk; security teams need evidence; end users need to see what's in it for them; finance needs numbers they can defend. AI makes communication harder: hype and fear distort expectations, and risks like hallucination or prompt injection are unfamiliar. Consultants who can tell a clear story, run convincing demos, handle objections honestly and explain AI risk in plain language move projects forward.

## Key concepts

### Know your audience

| Stakeholder | Cares about | Communicate with |
|---|---|---|
| **Executive sponsor / C-suite** | Strategic value, risk, cost, timeline, reputation | One-page summaries, outcomes, decisions needed |
| **Business owners** | Process improvement, KPIs, adoption, team impact | Process before/after, metrics, pilot results |
| **IT/architecture** | Fit, standards, operability, integration | Designs, ADRs, reference architectures |
| **Security and compliance** | Threats, controls, data protection, regulation | Threat models, control mappings, evidence |
| **Finance** | Cost, ROI, predictability | Business case, scenarios, FinOps plan |
| **Legal / privacy / works councils** | Liability, data use, employee impact | Impact assessments, data flows, transparency |
| **End users** | Ease, usefulness, job impact | Demos, training, feedback channels |

### Executive storytelling

A simple structure: **Situation** (context and pain) → **Complication** (why now, cost of inaction) → **Resolution** (recommendation, value, investment, risks, decision needed). Lead with the answer; use numbers sparingly and credibly; one key message per slide; end with a clear ask.

### Demos that build trust

| Practice | Why |
|---|---|
| Use realistic scenarios and client-like data | Relevance |
| Show limits and safeguards (citations, refusals, human approval) | Credibility and trust |
| Rehearse and have a recorded backup | Live AI demos can fail |
| Keep it short (5–10 minutes) and tied to the business outcome | Attention |
| Show the "before" and "after" process | Value is visible |

### Handling objections

| Objection | Response approach |
|---|---|
| "AI makes things up" | Explain grounding, evaluation metrics, citations, human review; share measured results |
| "Our data will train someone's model" | Explain Microsoft data commitments (Topic 23 file 07) and controls |
| "It's too expensive" | Business case scenarios, cost controls, phased approach |
| "Security won't allow it" | Show landing zone, private networking, guardrails, Defender, threat model |
| "Our people will resist" | Change management plan, champions, augmentation framing |
| "Why not wait?" | Cost of delay, competitive context, low-risk pilot |

Acknowledge valid concerns; never dismiss them.

### Explaining AI risk plainly

| Risk | Plain-language explanation |
|---|---|
| Hallucination | "It can produce confident but wrong answers. We reduce this by grounding answers in your documents, measuring accuracy, showing sources and keeping people in the loop for important decisions." |
| Prompt injection | "Someone can hide instructions in a document or email to trick the assistant. We limit what it can do, scan inputs and require approval for sensitive actions." |
| Data leakage | "It can only show what each user is already allowed to see, and we've cleaned up overshared files first." |
| Bias | "We test results across groups and languages and keep humans accountable for decisions." |

### Communication cadence

Kick-off, weekly status (RAG status, progress, risks, decisions needed), steering committee monthly, milestone readouts, demos at sprint ends, executive summary at decision gates.

## How it works on Azure

Evidence you can show from Azure:

| Concern | Evidence |
|---|---|
| Quality | Foundry evaluation results and trends |
| Safety | Red teaming scorecards, guardrail configuration |
| Security | Secure Score, Defender recommendations, private endpoint topology |
| Cost | Cost Management dashboards, budgets, forecast |
| Reliability | SLO dashboards, failover test results |
| Adoption | Copilot Dashboard, Copilot Studio analytics, app telemetry |

Status report template (one page): overall status, achievements this period, next period, key metrics, risks/issues with owners, decisions required.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Detail level** | Executives: outcomes and decisions; technical teams: depth and evidence. |
| **Honesty vs enthusiasm** | Be optimistic about value and candid about risk; credibility compounds. |
| **Live vs recorded demo** | Live for engagement, recorded backup for safety. |
| **Written vs verbal** | Pre-reads for complex decisions; meetings for discussion. |

## Hands-on

1. Write a one-page executive summary for an AI pilot using situation–complication–resolution.
2. Prepare and rehearse a seven-minute demo with a recorded backup.
3. Draft responses to five common objections with evidence.
4. Explain three AI risks in plain language to a non-technical colleague.
5. Create a one-page status report template.

## Security, cost and reliability notes

- **Don't use confidential client data in demos** shown to wider audiences without approval.
- **Avoid overpromising** on accuracy, savings or timelines.
- **Transparency** about incidents and risks builds long-term trust.

## Common mistakes

| Mistake | Fix |
|---|---|
| Technical deep dives for executives | Outcome-first summaries |
| Demos without safeguards shown | Show citations, refusals, approvals |
| Dismissing concerns | Acknowledge and evidence |
| Status reports hiding red items | Transparent RAG status |
| No clear ask | End with the decision needed |

## Check yourself

1. What does each stakeholder group care about?
2. Describe the situation–complication–resolution structure.
3. List five demo practices.
4. How would you respond to "AI makes things up"?
5. Explain prompt injection in plain language.
6. What Azure evidence supports stakeholder communication?

<details>
<summary>Answers</summary>

1. Executives: value, risk, cost; business owners: process and KPIs; IT: fit and operability; security: controls and evidence; finance: ROI; legal: liability and data use; users: usefulness and job impact.
2. Context and pain, why now/cost of inaction, recommendation with value, investment, risks and decision needed.
3. Realistic scenarios, show limits and safeguards, rehearse with recorded backup, keep short and outcome-focused, show before/after.
4. Explain grounding, evaluation metrics, citations and human review, and share measured results.
5. Hidden instructions in content can try to trick the assistant; we limit its actions, scan inputs and require approvals.
6. Evaluation results, red teaming scorecards, Secure Score and Defender, cost dashboards, SLO dashboards, adoption analytics.

</details>

## References

- [Data, privacy and security for Models sold by Azure](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)
- [AI Red Teaming Agent](https://learn.microsoft.com/azure/foundry/concepts/ai-red-teaming-agent)
- [Microsoft 365 Copilot Dashboard](https://learn.microsoft.com/viva/insights/org-team-insights/copilot-dashboard)
- [Responsible AI for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/responsible-use-of-ai-overview)
