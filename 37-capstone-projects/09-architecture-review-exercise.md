# Architecture Review Exercise

## Why it matters

Architects and consultants spend as much time reviewing designs as creating them. A good review finds the gaps that cause outages, breaches and cost overruns. It prioritises them by risk, and it writes recommendations a team can act on. This capstone gives you a **provided design** with deliberate weaknesses. You run a **Well-Architected review** and write the findings report: a deliverable you'll produce many times in client work.

## Key concepts

### The provided design: "Contoso Claims Copilot"

```text
Users (internet) ─▶ App Service (single instance, public, B1 plan, UK South)
                     ├─ connection strings and model API key in app settings
                     ├─▶ Azure OpenAI / Foundry deployment (standard, UK South, key auth, public network access)
                     ├─▶ Azure AI Search (Basic, 1 replica, public)
                     ├─▶ Azure SQL Database (General Purpose, no zone redundancy, public endpoint + "Allow Azure services")
                     └─▶ Blob Storage (LRS, public network, shared key enabled) — claim documents with personal data
Monitoring: default App Service metrics only. No alerts.
Deployment: developers publish from Visual Studio.
Business context: tier-1 customer-facing claims assistant, 50,000 users, target SLO 99.9%, RTO 4h, RPO 15 min,
                  UK GDPR applies, budget £8,000/month, launch in 8 weeks.
AI behaviour: system prompt only; no content filters configured beyond default; no evaluation; answers cite nothing;
              the assistant can "update claim status" via a function that calls SQL with the app's credentials.
```

### Review method

1. **Clarify** context: business criticality, SLOs, compliance, budget, team skills.
2. **Assess** each pillar with the Azure Well-Architected Review (core and AI workload perspectives) plus service guides.
3. **Gather evidence**: Advisor, Defender for Cloud, Policy compliance, cost data (for the exercise, reason from the design).
4. **Rate** each finding by risk (likelihood × impact) and effort.
5. **Recommend** specific changes with trade-offs.
6. **Prioritise** into a roadmap: must fix before launch, next 90 days, later.

### Skills practised

Topics [33 files 01–02](../33-solution-architecture/02-well-architected-reviews-and-service-guides.md), [34](../34-ai-solution-architecture/README.md), [25](../25-security/README.md), [28](../28-resiliency-backup-and-disaster-recovery/README.md), [29](../29-cost-management-and-finops/README.md) and [35 file 07](../35-consulting-practice/07-solution-design-deliverables.md).

## How it works on Azure

### Findings report structure

| Section | Content |
|---|---|
| Executive summary | Overall readiness, top five risks, launch recommendation |
| Context and scope | Workload, requirements, assumptions, evidence used |
| Findings by pillar | ID, finding, risk rating, evidence, recommendation, effort, owner |
| AI-specific findings | Grounding, evaluation, safety, agent tool permissions, model capacity |
| Prioritised roadmap | Pre-launch, 90 days, later |
| Target architecture | Revised diagram |
| Cost impact | Before and after estimate against budget |

### Example finding format

| ID | Pillar | Finding | Risk | Recommendation | Effort |
|---|---|---|---|---|---|
| SEC-01 | Security | Model and storage accessed with keys stored in app settings | High | Managed identity with RBAC; disable local auth; Key Vault for any remaining secrets | Low |
| REL-01 | Reliability | Single-instance B1 App Service can't meet 99.9% and has no zone redundancy | High | Premium plan with zone redundancy and at least 3 instances; health checks | Medium |

Write at least **20 findings** across the five pillars and the AI-specific area, without looking at the hints below first.

<details>
<summary>Hints (open after your first pass)</summary>

- **Security:** public endpoints everywhere; key auth; shared key on storage holding personal data; SQL "Allow Azure services"; the update-claim function uses the app's broad credentials with no user authorisation; no Defender plans; no prompt shields.
- **Reliability:** single instance; no zones; SQL tier and backup configuration against RPO 15 min; AI Search with 1 replica (no read SLA); single region model with no fallback; no retry or back-off on 429s.
- **Cost:** no budgets; no token monitoring; model choice not evaluated; possible right-sizing.
- **Operational excellence:** manual deployments; no IaC; no alerts; no tracing; no runbooks.
- **Performance:** no load testing; no quota planning against 50,000 users; no caching.
- **AI-specific:** no evaluation; no citations; no groundedness checks; default safety only; write action without approval; no responsible AI impact assessment; personal data in prompts without DPIA.

</details>

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Launch recommendation | "Go with conditions" is valid if pre-launch must-fixes are achievable in 8 weeks |
| Budget tension | Show the cost of each reliability and security improvement; propose phasing |
| Depth vs breadth | Cover all pillars, then go deep on the top risks |
| Tone | Factual and constructive; recognise what's done well |

## Hands-on

1. Write the full findings report (5–10 pages).
2. Draw the revised target architecture.
3. Estimate monthly cost before and after against the £8,000 budget.
4. Prepare a 10-minute executive readout with a launch recommendation.
5. Compare your findings with a peer's and reconcile differences.

## Security, cost and reliability notes

- Prioritise findings that combine high impact and low effort (keys, public access, alerts).
- Personal data and UK GDPR make security and privacy findings launch blockers.
- Reliability recommendations must be justified by the 99.9% SLO and the RTO and RPO, not by best practice alone.

## Common mistakes

| Mistake | Fix |
|---|---|
| Listing best practices without linking them to requirements | Tie each finding to a requirement or risk |
| No prioritisation | Risk × effort roadmap |
| Ignoring AI-specific risks | Add an AI workload perspective |
| Recommending everything | Fit the budget and timeline |

## Check yourself

1. What steps make up a Well-Architected review?
2. Which five findings would you treat as launch blockers?
3. How do you rate and prioritise findings?
4. Why is the update-claim function a serious risk?
5. What belongs in the executive summary?

<details>
<summary>Answers</summary>

1. Clarify context, assess each pillar, gather evidence, rate risks, recommend changes, prioritise into a roadmap.
2. For example: key-based and public access to personal data, write action without user authorisation or approval, single-instance hosting against a 99.9% SLO, no evaluation or groundedness checks, no monitoring or alerts.
3. Likelihood × impact for risk, plus effort; then group into pre-launch, 90 days and later.
4. The model can trigger data changes with broad app credentials and no user-level authorisation or human approval, so prompt injection or errors could change claims.
5. Overall readiness, top risks, launch recommendation with conditions, and the cost and time to remediate.

</details>

## References

- [Azure Well-Architected Review assessment](https://learn.microsoft.com/assessments/azure-architecture-review/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [AI workloads on Azure (Well-Architected)](https://learn.microsoft.com/azure/well-architected/ai/get-started)
- [Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
