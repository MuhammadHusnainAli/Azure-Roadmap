# Security, Data and DevOps Certifications

## Why it matters

AI solutions sit on top of security, data and delivery platforms, and specialists in those areas are often the people who approve or block AI work. A certification in one adjacent area adds credibility and widens the roles open to you. A security certification helps you pass security reviews. A Fabric certification helps you own the data that feeds RAG and analytics. AZ-400 proves you can build the pipelines that GenAIOps depends on. This file summarises the adjacent certifications most relevant to the roadmap's three tracks.

## Key concepts

### At a glance

| Exam | Certification | Level | Prerequisite | Best for |
|---|---|---|---|---|
| **SC-500** | Cloud and AI Security Engineer Associate | Associate | None | Engineers securing Azure and AI workloads |
| **SC-100** | Cybersecurity Architect Expert | Expert | One of SC-200, SC-300 or SC-500 Associates | Security-led architects and consultants |
| **DP-700** | Fabric Data Engineer Associate | Associate | None | Data pipelines for analytics and AI |
| **DP-600** | Fabric Analytics Engineer Associate | Associate | None | Semantic models and analytics solutions |
| **AZ-400** | DevOps Engineer Expert | Expert | Azure Administrator or Azure Developer Associate (check page) | Platform and pipeline engineers |
| **AZ-700** | Azure Network Engineer Associate | Associate | None | Network-heavy AI and landing zone work |

### SC-500 Cloud and AI Security Engineer

SC-500 replaced AZ-500, which retired on 31 August 2026. SC-500 became available on 21 July 2026, and its scope extends from Azure security into AI workload security.

| Domain | Weight |
|---|---|
| Manage identity, access and governance | 20–25% |
| Secure storage, databases and networking | 25–30% |
| Secure compute | 20–25% |
| Manage and monitor security posture | 20–25% |

Roadmap: [25 · Security](../25-security/README.md), [23 · Responsible AI, Security and Governance](../23-responsible-ai-security-and-governance/README.md), [02 · Identity](../02-identity-and-access/README.md), [05 · Networking](../05-networking/README.md).

### SC-100 Cybersecurity Architect

The study guide was updated on 21 October 2026. Its domains are:

| Domain | Weight |
|---|---|
| Design solutions that align with security best practices and priorities | 20–25% |
| Design security operations, identity and compliance capabilities | 25–30% |
| Design security solutions for infrastructure | 25–30% |
| Design security solutions for applications and data | 20–25% |

Roadmap: topics 02, 03, 23, 25 and 33. Covers Zero Trust, the Microsoft Cybersecurity Reference Architectures and the Microsoft cloud security benchmark.

### DP-700 Fabric Data Engineer and DP-600 Fabric Analytics Engineer

| Exam | Domains (weights) |
|---|---|
| **DP-700** | Implement and manage an analytics solution (30–35%); ingest and transform data (30–35%); monitor and optimise an analytics solution (30–35%) |
| **DP-600** | Plan, implement and manage a data analytics solution (10–15%); prepare and serve data (40–45%); implement and manage semantic models (20–25%); explore and analyse data (20–25%) |

Roadmap: [12 · Data and Analytics](../12-data-and-analytics/README.md), with [17 · RAG and Knowledge](../17-rag-and-knowledge/README.md) for the AI angle (Fabric IQ, OneLake as a knowledge source).

### AZ-400 DevOps Engineer Expert

The study guide was updated on 27 July 2026. Its domains are:

| Domain | Weight |
|---|---|
| Processes and communications | 10–15% |
| Source control strategy | 10–15% |
| Build and release pipelines | 50–55% |
| Security and compliance plan | 10–15% |
| Instrumentation strategy | 5–10% |

The prerequisite list on the certification page names the Azure Administrator Associate and the Azure Developer Associate, which has retired. Check whether the Azure AI Cloud Developer Associate (AI-200) has been added. Roadmap: [04](../04-tooling-and-infrastructure-as-code/README.md), [27 · DevOps and Platform Engineering](../27-devops-and-platform-engineering/README.md), [22](../22-evaluation-observability-and-genaiops/README.md).

### AZ-700 Network Engineer

The study guide was updated on 27 July 2026. Its domains are:

| Domain | Weight |
|---|---|
| Core networking infrastructure | 25–30% |
| Connectivity services | 20–25% |
| Application delivery services | 15–20% |
| Private access to Azure services | 10–15% |
| Network security services | 15–20% |

Roadmap: [05 · Networking](../05-networking/README.md), [34 file 06 · Network-Isolated AI Architecture](../34-ai-solution-architecture/06-network-isolated-ai-architecture.md).

## How it works on Azure

### Which adjacent certification for which role?

| Role or situation | Recommended |
|---|---|
| AI engineer who keeps hitting security reviews | SC-500 |
| Consultant leading regulated-industry AI work | SC-100 (after SC-500 or SC-300) |
| AI engineer owning RAG data pipelines | DP-700 |
| Analytics-heavy AI or Copilot-for-data projects | DP-600 |
| Platform engineer building GenAIOps | AZ-400 |
| Architect designing private AI landing zones | AZ-700 |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **One adjacent certification, not five** | Pick the one that removes your biggest delivery blocker. |
| **Security Associate choice** | SC-500 for Azure and AI workloads; SC-200 for security operations; SC-300 for identity. |
| **Fabric vs Azure data** | The data certifications are now Fabric-centred; Azure data services appear in AZ-305 and AI-200. |

## Hands-on

1. Map your last project's blockers to one adjacent certification.
2. Secure a Foundry project end to end (private endpoints, managed identity, Defender for AI) as SC-500 practice.
3. Build a Fabric pipeline that lands documents in OneLake for indexing.
4. Build a multi-stage pipeline with approvals and environment protections for AZ-400.
5. Design hub-spoke with Firewall and private DNS for AI services for AZ-700.

## Security, cost and reliability notes

- Security exams test Defender, Sentinel and Purview features that may need paid licences. Use trials carefully.
- Fabric trials and capacities cost money if left running; pause capacities.
- Network labs (Firewall, ExpressRoute simulators, Application Gateway) are expensive; delete them promptly.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying AZ-500 in late 2026 | Use SC-500 |
| Assuming DP-203 still exists | Data engineering moved to DP-700 (Fabric) |
| Booking SC-100 without a prerequisite | Earn SC-200, SC-300 or SC-500 first |
| Collecting certifications without a plan | Choose by role and blocker |

## Check yourself

1. What replaced AZ-500, and when was it available?
2. Which certifications satisfy the SC-100 prerequisite?
3. Which domain dominates AZ-400?
4. What's the difference in focus between DP-700 and DP-600?
5. Which adjacent certification suits an architect designing private AI landing zones?

<details>
<summary>Answers</summary>

1. SC-500, Cloud and AI Security Engineer Associate, available from 21 July 2026.
2. Identity and Access Administrator (SC-300), Security Operations Analyst (SC-200) or Cloud and AI Security Engineer (SC-500) Associate.
3. Design and implement build and release pipelines (50–55%).
4. DP-700 covers data engineering (ingest, transform, operate); DP-600 covers analytics (prepare and serve data, semantic models, analysis).
5. AZ-700 Azure Network Engineer Associate.

</details>

## References

- [Cloud and AI Security Engineer Associate](https://learn.microsoft.com/credentials/certifications/cloud-and-ai-security-engineer-associate/)
- [SC-500 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/sc-500)
- [Cybersecurity Architect Expert](https://learn.microsoft.com/credentials/certifications/cybersecurity-architect-expert/)
- [Fabric Data Engineer Associate](https://learn.microsoft.com/credentials/certifications/fabric-data-engineer-associate/)
- [Fabric Analytics Engineer Associate](https://learn.microsoft.com/credentials/certifications/fabric-analytics-engineer-associate/)
- [DevOps Engineer Expert](https://learn.microsoft.com/credentials/certifications/devops-engineer/)
- [Azure Network Engineer Associate](https://learn.microsoft.com/credentials/certifications/azure-network-engineer-associate/)
