# Well-Architected Reviews and Service Guides

## Why it matters

Architectures drift: features are added, services change, incidents reveal weaknesses, and nobody re-reads the original design. A **Well-Architected Review (WAR)** is a structured assessment of a workload against the five pillars that produces a prioritised list of improvements. Done well, it's one of the most valuable engagements an architect or consultant can run: a few days of work that surfaces resilience gaps, security risks and savings before they become incidents or invoices. Microsoft provides the **Azure Well-Architected Review** assessment on Microsoft Learn, **service guides** with per-service recommendations, **Azure Advisor** signals, and partner programmes for formal reviews.

## Key concepts

### Review inputs

| Input | Source |
|---|---|
| Business context | Criticality, users, SLOs, compliance, budget |
| Architecture | Diagrams, IaC, ADRs, data flows, dependencies |
| Telemetry | Availability, latency, errors, incidents, capacity |
| Platform signals | Azure Advisor (by pillar), Defender for Cloud Secure Score and recommendations, Policy compliance, Cost Management, Resource Graph inventory, Service Health history |
| People and process | Deployment process, on-call, runbooks, DR tests, security practices |

### Review process

| Step | Activities |
|---|---|
| **1. Prepare** | Scope the workload, agree pillars and depth, collect documentation and access (Reader + Advisor/Defender/Cost views) |
| **2. Assess** | Run the Well-Architected Review assessment questionnaire with the team; inspect configuration against service guides; review telemetry |
| **3. Analyse** | Identify risks and gaps; rate impact and effort; consider trade-offs |
| **4. Recommend** | Prioritised backlog: quick wins, strategic changes, accepted risks |
| **5. Report** | Findings, evidence, recommendations, owners, timeline; executive summary |
| **6. Follow up** | Track remediation; re-assess after changes; milestones in the assessment tool |

### Finding format

| Field | Example |
|---|---|
| Pillar | Reliability |
| Finding | SQL Database not zone redundant; single region |
| Evidence | Resource config, Advisor recommendation |
| Risk | Zone outage causes full outage (RTO hours) |
| Recommendation | Enable zone redundancy; add failover group to secondary region for RTO < 1 h |
| Impact / effort | High / Medium |
| Trade-off | +€X/month |
| Owner / target date | DBA team / Q1 |

### Service guides

Each service guide lists recommendations per pillar for one service (for example AKS: availability zones, cluster autoscaler, Azure CNI choices, workload identity, Defender for Containers, upgrade strategies, cost analysis add-on). Use them as checklists during reviews and design.

### Assessment tool

The **Azure Well-Architected Review** on Microsoft Learn Assessments guides questions per pillar (and workload types such as AI or mission-critical), scores the workload, generates recommendations with links, and supports **milestones** to track progress over time. Results can be exported.

## How it works on Azure

Evidence collection with Resource Graph (examples):

```kusto
// Resources without zone redundancy flags (example for SQL databases)
resources
| where type =~ "microsoft.sql/servers/databases" and name != "master"
| extend zr = tobool(properties.zoneRedundant)
| project name, resourceGroup, sku = tostring(sku.name), zr

// Public network access enabled on PaaS resources
resources
| where isnotempty(properties.publicNetworkAccess) and tostring(properties.publicNetworkAccess) =~ "Enabled"
| project name, type, resourceGroup
```

Advisor recommendations by category (Topic 26 file 08) and Defender recommendations by severity (Topic 25 file 02) feed the findings.

Report structure:

1. Executive summary (overall posture, top five risks, investment needed).
2. Scope and method.
3. Findings by pillar with evidence.
4. Prioritised roadmap (now / next / later).
5. Accepted risks and trade-offs.
6. Appendices (inventory, assessment export, queries).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Depth** | Light review (questionnaire + Advisor) for low-criticality; deep review with configuration inspection and testing for critical workloads. |
| **Who participates** | Workload team, operations, security, product owner; independent reviewer for objectivity. |
| **Prioritisation** | Risk × likelihood vs effort; regulatory items first. |
| **Frequency** | Annually for most; after major changes or incidents; quarterly for mission-critical. |
| **Automation** | Automate evidence collection (Resource Graph, Advisor, Policy) to make reviews cheaper and repeatable. |

## Hands-on

1. Run the Azure Well-Architected Review assessment for a workload you know.
2. Collect evidence with the Resource Graph queries, Advisor and Defender.
3. Write five findings in the finding format.
4. Build a now/next/later roadmap and estimate the cost of recommendations.
5. Record a milestone and plan a follow-up review.

## Security, cost and reliability notes

- **Access for reviewers**: read-only, time-bound (PIM); avoid sharing secrets in review materials.
- **Sensitive findings**: security gaps in reports need controlled distribution.
- **Cost of remediation**: present alongside risk reduction.

## Common mistakes

| Mistake | Fix |
|---|---|
| Questionnaire only, no evidence | Inspect configuration and telemetry |
| Hundreds of findings with no priority | Risk-based prioritisation |
| No owners or dates | Assign and track |
| Review once, never again | Milestones and re-assessments |
| Ignoring trade-offs and cost | Include them in recommendations |

## Check yourself

1. What inputs does a Well-Architected Review need?
2. Describe the review process steps.
3. What fields should a finding contain?
4. How do service guides help?
5. What does the assessment tool provide?
6. How often should reviews happen?

<details>
<summary>Answers</summary>

1. Business context, architecture documentation, telemetry, platform signals (Advisor, Defender, Policy, Cost, Resource Graph, Service Health) and process information.
2. Prepare, assess, analyse, recommend, report, follow up.
3. Pillar, finding, evidence, risk, recommendation, impact/effort, trade-off, owner and target date.
4. They provide per-service recommendations by pillar to use as checklists.
5. Guided questions per pillar and workload type, scoring, linked recommendations, milestones and export.
6. Annually for most workloads, after major changes or incidents, and quarterly for mission-critical ones.

</details>

## References

- [Azure Well-Architected Review assessment](https://learn.microsoft.com/assessments/azure-architecture-review/)
- [Service guides](https://learn.microsoft.com/azure/well-architected/service-guides/)
- [Implementing recommendations](https://learn.microsoft.com/azure/well-architected/cross-cutting-guides/implementing-recommendations)
- [Azure Advisor](https://learn.microsoft.com/azure/advisor/advisor-overview)
