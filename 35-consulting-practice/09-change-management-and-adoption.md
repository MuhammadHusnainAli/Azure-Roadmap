# Change Management and Adoption

## Why it matters

A technically perfect AI solution that people don't use delivers zero value. Adoption is usually the biggest variable in AI business cases: Microsoft 365 Copilot licences sit idle, agents are tried once and abandoned, users don't trust outputs, managers don't change processes, and employees worry about their jobs. **Change management** prepares people, processes and organisations for new ways of working; **adoption programmes** drive and measure usage and value. Consultants who plan adoption from day one (not as an afterthought) turn pilots into lasting change.

## Key concepts

### Change frameworks

| Element | Practice |
|---|---|
| **Sponsorship** | Visible executive sponsor who uses the tool and explains why |
| **Awareness and desire** | Communicate the why, the what's-in-it-for-me, and address fears (ADKAR-style: Awareness, Desire, Knowledge, Ability, Reinforcement) |
| **Knowledge and ability** | Role-based training, prompt skills, scenario libraries, hands-on practice |
| **Reinforcement** | Recognition, success stories, embedding in processes and KPIs |
| **Impacted groups** | Analyse how each role's work changes |

### Adoption programme components

| Component | Detail |
|---|---|
| **Champions network** | Enthusiastic users in each team who share tips, collect feedback and support peers |
| **Scenario library** | Role-specific use cases and prompts ("draft a customer reply", "summarise this RFP") |
| **Training** | Short, role-based, practical; office hours; self-service resources (Microsoft Adoption hub, Copilot enablement resources) |
| **Communications** | Launch campaigns, newsletters, success stories, leadership messages |
| **Feedback loops** | In-product feedback, surveys, champion input feeding the backlog |
| **Process redesign** | Change procedures, approvals and KPIs so AI use becomes the norm |
| **Responsible use guidance** | When to rely on AI, when to verify, what not to share |
| **Measurement** | Usage, engagement, sentiment, outcome metrics |

### Measuring adoption

| Metric | Source |
|---|---|
| Active users, usage frequency, retention | Copilot Dashboard (Viva Insights), Microsoft 365 admin reports, app telemetry |
| Feature/agent usage | Copilot Studio analytics, Agent 365 analytics, Application Insights |
| Satisfaction and sentiment | Surveys, thumbs up/down |
| Outcome impact | Process metrics (cycle time, handling time, quality) (Topic 22 file 10) |
| Licence utilisation | Assigned vs active licences |

### Workforce considerations

Be transparent about impact on roles; frame as augmentation where true; involve HR and, where applicable, works councils early; provide reskilling paths; respect monitoring and privacy rules when measuring usage (aggregate, team-level).

## How it works on Azure

Adoption plan for an internal AI assistant (12 weeks):

```text
Weeks 1–2: stakeholder and impact analysis; sponsor message; recruit champions (1 per 25–50 users)
Weeks 3–4: scenario library and training materials; pilot group onboarding; feedback channel in Teams
Weeks 5–8: pilot usage; weekly champion syncs; measure usage and satisfaction; fix top issues
Weeks 9–10: process changes (templates, SOPs referencing the assistant); success stories
Weeks 11–12: broad rollout in waves; training at scale; adoption dashboard to leadership
Ongoing: monthly review of usage, outcomes, feedback; refresh scenarios; recognise champions
```

Data sources: Copilot Dashboard in Viva Insights for Microsoft 365 Copilot and agents; Copilot Studio analytics; Application Insights custom events for custom apps (feature usage, thumbs up/down); Power BI adoption report combining these with business KPIs.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Wave rollout** | Start with teams that have clear use cases and strong champions. |
| **Licence allocation** | Assign Copilot licences based on demonstrated use cases; reallocate unused licences. |
| **Training depth** | Short and scenario-based beats long generic training. |
| **Measurement privacy** | Aggregate team-level metrics; avoid individual surveillance. |
| **Incentives** | Recognition and process integration over mandates. |

## Hands-on

1. Run an impact analysis for three roles affected by an AI assistant.
2. Build a scenario library with ten role-specific use cases.
3. Design a champions programme (selection, responsibilities, cadence).
4. Create an adoption dashboard from available telemetry.
5. Write launch communications from the sponsor.

## Security, cost and reliability notes

- **Responsible use guidance** reduces data leakage and over-reliance risks.
- **Unused licences** waste money; track utilisation.
- **Reliability**: early outages destroy trust; stabilise before broad rollout.

## Common mistakes

| Mistake | Fix |
|---|---|
| "Build it and they will come" | Planned adoption programme |
| Generic training | Role-based scenarios |
| No champions | Champions network |
| Measuring only logins | Outcome and sentiment metrics |
| Ignoring job concerns | Transparent communication and HR involvement |

## Check yourself

1. What does ADKAR stand for?
2. List the components of an adoption programme.
3. What does a champions network do?
4. Which metrics measure adoption?
5. Which Microsoft tools provide adoption data?
6. How should workforce concerns be handled?

<details>
<summary>Answers</summary>

1. Awareness, Desire, Knowledge, Ability, Reinforcement.
2. Champions, scenario library, training, communications, feedback loops, process redesign, responsible use guidance, measurement.
3. Shares tips, supports peers, collects feedback and promotes adoption within teams.
4. Active users, frequency, retention, feature/agent usage, satisfaction, outcome metrics, licence utilisation.
5. Copilot Dashboard (Viva Insights), Microsoft 365 admin reports, Copilot Studio analytics, Agent 365 analytics, Application Insights.
6. Transparency, augmentation framing where true, early HR/works council involvement, reskilling and privacy-respecting measurement.

</details>

## References

- [Microsoft Adoption hub](https://adoption.microsoft.com/)
- [Microsoft 365 Copilot enablement resources](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-enablement-resources)
- [Copilot Dashboard (Viva Insights)](https://learn.microsoft.com/viva/insights/org-team-insights/copilot-dashboard)
- [Copilot Studio analytics](https://learn.microsoft.com/microsoft-copilot-studio/analytics-overview)
- [CAF Adopt](https://learn.microsoft.com/azure/cloud-adoption-framework/adopt/)
