# AZ-305 Azure Solutions Architect Expert

## Why it matters

AZ-305 (Designing Microsoft Azure Infrastructure Solutions) leads to **Microsoft Certified: Azure Solutions Architect Expert**, the flagship Azure architecture credential. It tests design judgement rather than configuration. You must choose identity, governance, monitoring, data, continuity, compute, application, migration and network designs that meet stated requirements and constraints. For the **AI Solution Architect** and **Azure Consultant** tracks it's the credential clients and employers look for first, and AI architectures still rest on these foundations. You must hold the **Azure Administrator Associate** (AZ-104) to earn the certification.

## Key concepts

### Skills measured (study guide, as of 17 April 2026)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Design identity, governance and monitoring solutions** | 25–30% | Logging and monitoring; authentication and authorisation; governance |
| **Design data storage solutions** | 20–25% | Relational data; semi-structured and unstructured data; data integration |
| **Design business continuity solutions** | 15–20% | Backup and disaster recovery; high availability |
| **Design infrastructure solutions** | 30–35% | Compute; application architecture; migrations; network solutions |

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| Identity, authorisation | [02 · Identity and Access](../02-identity-and-access/README.md), [25 · Security](../25-security/README.md) |
| Governance | [03 · Governance and Landing Zones](../03-governance-and-landing-zones/README.md), [29 · Cost Management](../29-cost-management-and-finops/README.md) |
| Logging and monitoring | [26 · Monitoring and Observability](../26-monitoring-and-observability/README.md) |
| Data storage and integration | [08 · Storage](../08-storage/README.md), [09 · Databases](../09-databases/README.md), [12 · Data and Analytics](../12-data-and-analytics/README.md) |
| Backup, DR, HA | [28 · Resiliency, Backup and DR](../28-resiliency-backup-and-disaster-recovery/README.md) |
| Compute, application architecture | [06](../06-compute/README.md), [07](../07-containers-and-kubernetes/README.md), [10](../10-application-platform/README.md), [11 · Integration](../11-integration-messaging-and-apis/README.md) |
| Network | [05 · Networking](../05-networking/README.md) |
| Migrations | [30 · Migration and Modernisation](../30-migration-and-modernisation/README.md) |
| IaC and DevOps context | [04](../04-tooling-and-infrastructure-as-code/README.md), [27](../27-devops-and-platform-engineering/README.md) |
| Design method and trade-offs | [33 · Solution Architecture](../33-solution-architecture/README.md) |

### How to think in AZ-305 questions

Each question gives requirements. Pick the option that meets **all** of them with the **least cost or administrative effort**. Train yourself to underline:

- **Availability and recovery targets** (SLA, RTO, RPO, zone or region failure).
- **Security and compliance constraints** (private connectivity, data residency, encryption keys).
- **Operational preferences** (PaaS over IaaS, minimise management).
- **Cost constraints** (minimise cost, use existing licences).

### Decision tables to know cold

| Decision | Options |
|---|---|
| Relational database | Azure SQL Database (single, elastic pool, Hyperscale), SQL Managed Instance, SQL Server on VMs, PostgreSQL/MySQL flexible server |
| Non-relational | Cosmos DB APIs and consistency levels, Blob tiers, Data Lake Storage |
| Compute | VMs, scale sets, App Service, Functions, Container Apps, AKS, Batch |
| Messaging | Service Bus vs Event Grid vs Event Hubs vs Storage queues |
| Connectivity | VPN vs ExpressRoute; hub-spoke vs Virtual WAN; private endpoints |
| Load balancing | Load Balancer, Application Gateway, Front Door, Traffic Manager |
| Identity | Entra ID, B2B, External ID, managed identities, Conditional Access, PIM |
| Continuity | Backup, Site Recovery, geo-replication, zone-redundant services |

## How it works on Azure

### Ten-week study plan (after AZ-104)

| Week | Focus | Practice |
|---|---|---|
| 1 | Method: WAF, requirements | Topic 33 files 01–03; write NFRs for a sample case |
| 2 | Identity and governance | Design management groups, policy and RBAC for a multi-subscription estate |
| 3 | Monitoring | Design log routing, workspaces and alerting for 50 subscriptions |
| 4 | Relational data | Compare SQL options for five scenarios |
| 5 | Non-relational and integration | Cosmos DB consistency choices; Data Factory vs Fabric pipelines |
| 6 | Continuity | RTO/RPO-driven DR design for three tiers of workloads |
| 7 | Compute and app architecture | Choose compute for six workloads; messaging patterns |
| 8 | Network | Hub-spoke with Firewall, ExpressRoute and private DNS at scale |
| 9 | Migrations | Azure Migrate assessment; database migration choices |
| 10 | Revision | Practice assessment; case studies; architecture review ([capstone 09](../37-capstone-projects/09-architecture-review-exercise.md)) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **AZ-305 vs AB-100** | AZ-305 for infrastructure-led architecture; AB-100 for agentic business solutions. Senior AI architects often hold both. |
| **Breadth vs depth** | AZ-305 rewards breadth: know when to use each service and the key trade-offs. |
| **Study by reading vs designing** | Design exercises beat reading. Write short ADRs for decisions. |

## Hands-on

1. Design a landing zone for a three-business-unit enterprise and justify each choice.
2. Produce a DR design with RTO and RPO for a tier-1 app, and estimate its cost.
3. Choose a database for five scenarios and write one-line justifications.
4. Run a Well-Architected review of an existing design.
5. Take the practice assessment; for each miss, write which requirement you overlooked.

## Security, cost and reliability notes

- Questions frequently hinge on **private connectivity**, **customer-managed keys** and **least privilege**.
- "Minimise cost" usually rules out over-engineered options such as multi-region active-active.
- Composite SLAs and zone or region redundancy are common reliability themes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Choosing what you'd personally build | Choose what meets the stated requirements |
| Ignoring a single constraint | Underline every requirement |
| Memorising without trade-offs | Learn why each option fits |
| Skipping AZ-104 | It's a prerequisite for the certification |

## Check yourself

1. What prerequisite does the Azure Solutions Architect Expert certification require?
2. Which AZ-305 domain carries the most weight?
3. How should you approach a question that says "minimise administrative effort"?
4. Name four load-balancing services and their scopes.
5. Which roadmap topic teaches the design method?

<details>
<summary>Answers</summary>

1. Microsoft Certified: Azure Administrator Associate (AZ-104).
2. Design infrastructure solutions (30–35%).
3. Prefer managed PaaS options and built-in features over self-managed infrastructure, as long as all other requirements are met.
4. Load Balancer (regional, layer 4), Application Gateway (regional, layer 7), Front Door (global, layer 7), Traffic Manager (global, DNS).
5. Topic 33 · Solution Architecture.

</details>

## References

- [Exam AZ-305](https://learn.microsoft.com/credentials/certifications/exams/az-305/)
- [AZ-305 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-305)
- [Azure Solutions Architect Expert](https://learn.microsoft.com/credentials/certifications/azure-solutions-architect/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
