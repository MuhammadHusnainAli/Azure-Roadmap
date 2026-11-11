# AI Readiness Assessment

## Why it matters

Many AI programmes stall not because the models fail but because the organisation isn't ready: data is scattered and over-shared, there's no secure platform to build on, security and legal teams block launches they weren't involved in, skills are thin, and governance doesn't exist until after an incident. An **AI readiness assessment** measures maturity across data, platform, security, skills and governance, identifies blockers for priority use cases, and produces a roadmap that makes AI adoption repeatable. It's one of the most requested consulting engagements and a natural follow-up to envisioning (file 02).

## Key concepts

### Readiness dimensions

| Dimension | What to assess | Evidence |
|---|---|---|
| **Strategy and value** | AI vision, sponsorship, prioritised use cases, business cases, value tracking | Strategy documents, use-case backlog |
| **Data** | Data availability, quality, cataloguing, access, permissions hygiene (oversharing), labelling, lineage, freshness, platform (lakehouse) | Purview Data Map/DSPM, data inventories, Fabric/Databricks estate |
| **Platform** | Landing zones, AI platform (Foundry, gateway, search), networking, identity, DevOps/GenAIOps, observability | Azure estate review (Resource Graph), architecture docs, pipelines |
| **Security** | Threat models for AI, guardrails, Defender for AI, data protection (Purview), identity for agents, secrets | Secure Score, Defender, policies, incident processes |
| **Governance and responsible AI** | Policies, intake and risk tiers, impact assessments, inventory of AI/agents, regulatory mapping | Policies, registers, Compliance Manager |
| **Skills and organisation** | AI engineering, data, architecture, product, change management; CoE; training plans | Skills inventory, org charts, certifications |
| **Operations** | Monitoring, evaluation, incident response for AI, cost management | Dashboards, runbooks, FinOps practices |
| **Culture and adoption** | Openness to change, champions, end-user readiness (for Copilot) | Surveys, adoption data |

### Maturity scale

| Level | Description |
|---|---|
| **1 – Initial** | Ad hoc experiments, no standards |
| **2 – Developing** | Some standards, isolated successes, gaps in governance/platform |
| **3 – Defined** | Platform, governance and processes established; repeatable delivery |
| **4 – Managed** | Measured outcomes, automation, scaling across business units |
| **5 – Optimising** | Continuous improvement, AI embedded in strategy and operations |

### Use-case-specific readiness

Overall maturity matters less than whether the **priority use cases** can proceed: for each, check data access and permissions, platform components needed, security/compliance requirements, skills and owners. Blockers become roadmap items.

### Microsoft resources

- **Cloud Adoption Framework for AI**: strategy, plan, ready (AI landing zones and platform architectures), govern, secure, manage.
- **Microsoft Learn Assessments**: AI readiness and cloud adoption assessments, Well-Architected Review.
- **Purview DSPM for AI**: oversharing and AI data risk signals (Topic 23 file 05).
- **Copilot readiness** resources for Microsoft 365 Copilot deployments (Topic 24 file 01).

## How it works on Azure

Assessment plan (3–4 weeks):

```text
Week 1: Kick-off, document review, stakeholder interviews (business, IT, data, security, legal, HR)
        Evidence collection: Resource Graph inventory, Secure Score and Defender, Policy compliance,
        Purview DSPM/oversharing assessment, Entra configuration, cost data, existing AI resources
Week 2: Workshops per dimension; Microsoft Learn AI readiness assessment with the team; WAF AI assessment for key workloads
Week 3: Scoring, gap analysis, use-case readiness checks, risk register
Week 4: Roadmap (0–3, 3–6, 6–12 months), quick wins, investment estimate, executive readout
```

Example findings and recommendations:

| Dimension | Finding | Recommendation |
|---|---|---|
| Data | 40% of SharePoint sites shared org-wide; no sensitivity labels | Oversharing remediation, label taxonomy, Restricted Content Discovery before Copilot/RAG |
| Platform | Teams call models directly with keys; no gateway | AI gateway with Entra auth and token limits |
| Security | No AI threat model; Defender for AI off | Threat modelling standard, enable Defender for AI, guardrail baseline |
| Governance | No inventory of AI/agents | Intake process, agent registry (Agent 365/Control Plane) |
| Skills | Two people with AI engineering experience | Training plan, hire/partner, CoE |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Breadth vs depth** | Organisation-wide maturity plus deep checks for 3–5 priority use cases. |
| **Evidence-based scoring** | Combine interviews with platform evidence to avoid optimism bias. |
| **Roadmap realism** | Sequence foundations (data hygiene, platform, governance) with visible quick wins. |
| **Ownership** | Each roadmap item has an owner and success measure. |
| **Re-assessment** | Repeat every 6–12 months to show progress. |

## Hands-on

1. Build an assessment questionnaire covering the eight dimensions with 5–8 questions each.
2. Collect Azure and Purview evidence for a test tenant.
3. Score maturity and produce a radar chart.
4. Check readiness for three use cases and list blockers.
5. Write a 12-month roadmap with quick wins and foundations.

## Security, cost and reliability notes

- **Sensitive findings** (security gaps, oversharing) require careful handling and limited distribution.
- **Access**: read-only, time-bound access for assessors.
- **Cost of remediation**: estimate and present alongside findings.

## Common mistakes

| Mistake | Fix |
|---|---|
| Interview-only assessments | Add platform evidence |
| Generic maturity scores with no use-case link | Use-case readiness checks |
| Ignoring data permissions | Oversharing assessment |
| Roadmaps without quick wins | Balance foundations with value |
| One-off assessment | Periodic re-assessment |

## Check yourself

1. List the eight readiness dimensions.
2. What evidence supports data and security readiness?
3. Describe the maturity scale.
4. Why assess use-case-specific readiness?
5. What does a typical assessment plan look like?
6. Which Microsoft resources support AI readiness assessments?

<details>
<summary>Answers</summary>

1. Strategy and value, data, platform, security, governance/responsible AI, skills and organisation, operations, culture and adoption.
2. Data: Purview Data Map/DSPM, inventories, permissions/oversharing. Security: Secure Score, Defender, policies, threat models, incident processes.
3. Initial, developing, defined, managed, optimising.
4. Because what matters is whether priority use cases can proceed; blockers become roadmap items.
5. Kick-off and evidence collection, dimension workshops and assessments, scoring and gap analysis, roadmap and readout over 3–4 weeks.
6. CAF for AI, Microsoft Learn Assessments (AI readiness, WAF), Purview DSPM for AI, Copilot readiness resources.

</details>

## References

- [AI adoption (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/)
- [Ready for AI: architecture and governance (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/ready)
- [Microsoft Learn Assessments](https://learn.microsoft.com/assessments/)
- [DSPM for AI](https://learn.microsoft.com/purview/dspm-for-ai)
- [Microsoft 365 Copilot enablement resources](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-enablement-resources)
