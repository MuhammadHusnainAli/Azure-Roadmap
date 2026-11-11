# Discovery and Envisioning Workshops

## Why it matters

The quality of an engagement is mostly decided in its first conversations. Discovery uncovers what the client actually needs (often different from what they asked for), who decides, what constraints exist and where value lies. Envisioning helps stakeholders imagine what's possible with cloud and AI and converge on a few high-value opportunities. Poorly run workshops produce wish lists and enthusiasm that evaporate; well-run ones produce a prioritised, owned backlog and momentum. This file gives consultants a repeatable method for stakeholder interviews, current-state assessment and AI use-case ideation.

## Key concepts

### Discovery components

| Component | Questions |
|---|---|
| **Business context** | Strategy, goals, pressures (cost, growth, regulation, competition), success measures |
| **Stakeholders** | Who sponsors, decides, influences, uses, operates? What does each care about? |
| **Current state: technology** | Estate (on-premises, Azure, other clouds), data platforms, identity, security posture, skills, existing AI |
| **Current state: process** | Pain points, manual work, volumes, cycle times, error rates, costs |
| **Constraints** | Budget, timelines, regulation, data residency, procurement, technology standards |
| **Readiness** | Data quality and access, governance, platform, skills, culture (file 04) |
| **Previous attempts** | What was tried, what worked, why things stalled |

### Stakeholder interviews

| Practice | Detail |
|---|---|
| Interview mix | Executive sponsor, business owners, IT/architecture, security/compliance, data, operations, end users |
| Format | 45–60 minutes, semi-structured, open questions, listen more than talk |
| Questions | "What would success look like in 12 months?", "Where do people spend time on low-value work?", "What keeps you up at night?", "What would stop this?" |
| Capture | Notes with themes, quotes, metrics, risks; validate back with interviewees |

### Current-state assessment on Azure

Use evidence, not only opinions: Resource Graph inventory, Advisor, Defender Secure Score, Policy compliance, Cost Management trends, Entra configuration, Purview/DSPM signals, existing AI usage (Foundry resources, Copilot adoption dashboards).

### Envisioning and AI use-case ideation

| Technique | Use |
|---|---|
| **Art of the possible** | Short demos and industry examples (Copilot, agents, document AI, analytics) to expand thinking |
| **Process walk-throughs** | Map a process step by step; identify where AI can assist, automate or augment |
| **Persona/day-in-the-life** | Pain points of roles (claims handler, engineer, salesperson) |
| **"How might we" prompts** | Turn pain points into opportunity statements |
| **Use-case canvas** | Problem, users, data, AI pattern, value, risks, feasibility, owner |
| **Dot voting and clustering** | Converge on top candidates |

AI pattern catalogue to guide ideation: knowledge assistants (RAG), document processing, customer/employee service agents, content generation, summarisation, classification/routing, forecasting/prediction, anomaly detection, coding assistance, analytics copilots.

## How it works on Azure

Two-day envisioning workshop agenda:

```text
Day 1
 09:00 Goals, success measures, ground rules (sponsor)
 09:30 Current state readout (from interviews and Azure evidence)
 10:30 Art of the possible: Copilot, Foundry agents, document AI, Fabric (demos)
 12:00 Lunch
 13:00 Process walk-throughs in groups (2–3 processes)
 15:00 Use-case ideation with canvas; "how might we"
 16:30 Clustering and first vote
Day 2
 09:00 Refine top 10 use cases (value, data, feasibility, risk)
 11:00 Scoring (value vs feasibility; file 03)
 13:00 Responsible AI and risk check for top 5
 14:00 Quick wins vs strategic bets; next steps (PoC, assessment, buy options)
 15:30 Readout to sponsor; owners and dates
```

Deliverable: discovery report (context, stakeholders, current state, findings, use-case backlog with canvases, prioritisation, recommended next steps, risks).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Workshop size** | 8–15 participants; mix business and technical; sponsor present at start and end. |
| **Pre-work** | Interviews and evidence collection before the workshop make sessions productive. |
| **Demos** | Short and relevant; avoid product pitches. |
| **Divergence vs convergence** | Allow broad ideation, then converge firmly with criteria. |
| **Remote vs in person** | In person for envisioning energy; remote works with strong facilitation and digital whiteboards. |

## Hands-on

1. Write an interview guide with 12 questions for executives and 12 for end users.
2. Run two practice interviews and synthesise themes.
3. Collect Azure evidence for a current-state readout (Resource Graph, Advisor, Secure Score, costs).
4. Facilitate a one-hour ideation session using the use-case canvas.
5. Produce a discovery report outline with the top five use cases.

## Security, cost and reliability notes

- **Confidential information** shared in workshops: agree NDAs and handling rules; don't paste client data into public AI tools.
- **Expectations**: avoid promising outcomes before feasibility is checked.
- **Bias**: include diverse voices (front-line users, risk, IT) to avoid blind spots.

## Common mistakes

| Mistake | Fix |
|---|---|
| Jumping to solutions | Understand problems and value first |
| Only talking to IT | Business and end users |
| Wish lists without owners | Owners and next steps |
| Demos that oversell | Realistic examples with limits |
| No follow-up | Readout and next steps within a week |

## Check yourself

1. What should discovery cover?
2. Who should be interviewed, and what makes a good interview?
3. How can Azure evidence support current-state assessment?
4. Name four ideation techniques.
5. What does a use-case canvas contain?
6. What should a discovery report include?

<details>
<summary>Answers</summary>

1. Business context, stakeholders, technology and process current state, constraints, readiness, previous attempts.
2. Sponsor, business owners, IT/architecture, security/compliance, data, operations, end users; semi-structured, open questions, active listening, validated notes.
3. Resource Graph inventory, Advisor, Secure Score, Policy compliance, cost trends, Entra configuration, Purview/DSPM, AI usage signals.
4. Any four: art of the possible demos, process walk-throughs, persona/day-in-the-life, "how might we", use-case canvas, dot voting.
5. Problem, users, data, AI pattern, value, risks, feasibility, owner.
6. Context, stakeholders, current state, findings, use-case backlog, prioritisation, recommendations, risks and next steps.

</details>

## References

- [CAF: plan for AI adoption](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/plan)
- [CAF: business plan for AI agents](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [CAF strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/strategy/)
- [Microsoft Adoption hub](https://adoption.microsoft.com/)
