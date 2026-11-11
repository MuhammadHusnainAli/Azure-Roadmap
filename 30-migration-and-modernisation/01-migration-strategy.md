# Migration Strategy

## Why it matters

Most large Azure programmes start with a migration: a datacentre exit, a hardware refresh, a VMware licensing change, an acquisition, or a push to modernise for AI. Programmes fail less often because of tooling than because of strategy: moving everything "as is" and paying more, modernising everything and never finishing, or migrating in the wrong order and breaking dependencies. Microsoft's Cloud Adoption Framework (CAF) gives a structured approach: decide a migration strategy per workload (the **Rs**), plan **waves** based on dependencies and business priorities, and treat migration as a repeatable factory with landing zones, governance and operations ready before the first workload moves.

## Key concepts

### Migration strategies (the Rs)

| Strategy | What changes | When to choose |
|---|---|---|
| **Retire** | Decommission | Unused, duplicate or low-value systems (often 10–20% of an estate) |
| **Retain** | Keep on-premises or where it is (for now) | Regulatory constraints, recent investment, imminent end-of-life, unsupported platforms |
| **Rehost** ("lift and shift") | Move VMs as-is to Azure IaaS or Azure VMware Solution | Fast exits, minimal change, deadline-driven |
| **Replatform** ("lift and optimise") | Small changes to use managed services (SQL MI, App Service, AKS) without redesign | Reduce operations with modest effort |
| **Refactor** | Modify code to fit PaaS/containers and cloud patterns | Improve scalability, maintainability, cost |
| **Rearchitect** | Significant redesign (microservices, event-driven, serverless) | Strategic apps needing agility and scale |
| **Rebuild** | Rewrite cloud-native | Legacy code can't evolve; high strategic value |
| **Replace** | Move to SaaS (Microsoft 365, Dynamics 365, third-party) | Commodity capabilities |

### Decision drivers

| Driver | Pushes towards |
|---|---|
| Deadline (datacentre exit, contract end) | Rehost, then optimise |
| Licensing changes (for example VMware) | Rehost to Azure VMs or Azure VMware Solution, then modernise |
| Operational burden | Replatform to PaaS |
| Innovation, AI integration | Refactor/rearchitect; modern data platform |
| Low business value | Retire or replace |
| Compliance constraints | Retain or sovereign options |

A common, pragmatic sequence: **rehost quickly where deadlines bite, replatform where easy, and modernise strategically afterwards** (file 08).

### CAF migration lifecycle

| Phase | Activities |
|---|---|
| **Strategy and plan** | Business outcomes, digital estate inventory, rationalisation (Rs), business case, skills plan |
| **Ready** | Azure landing zones, identity, networking, governance, operations baseline |
| **Migrate** | Waves: assess, migrate, test, cut over, decommission |
| **Govern/manage/secure** | Policies, monitoring, backup, security posture applied from day one |
| **Optimise** | Rightsizing, commitments, modernisation |

### Wave planning

A **wave** is a group of workloads migrated together, sized for the factory's capacity. Group by:

- **dependencies** (applications that talk constantly move together; dependency analysis in Azure Migrate, file 02);
- **business priority and risk** (pilot with low-risk workloads, then learn);
- **technology pattern** (similar workloads reuse runbooks);
- **downtime windows and business calendars** (avoid month-end, peak seasons);
- **shared data** (databases used by many apps).

Typical sequence: wave 0 (landing zone and pilot), early waves (simple, low-risk), main waves (bulk rehost/replatform), complex waves (mainframe, ERP, highly coupled systems).

### Success measures

Workloads migrated vs plan, cost vs business case, incidents during cutover, decommissioned on-premises capacity, post-migration performance, time per workload (factory velocity).

## How it works on Azure

Migration factory operating model:

```text
Discover (Azure Migrate appliance/collector) ─► Assess (readiness, sizing, cost, dependencies) ─► Rationalise (Rs per app with owners)
  ─► Wave plan ─► Landing zone ready (policies, networking, identity, monitoring, backup)
  ─► Migrate wave: replicate ─► test migration ─► business sign-off ─► cutover ─► hypercare ─► decommission source
  ─► Optimise and modernise backlog
```

Rationalisation table per application (stored in a migration tracker or Azure Migrate application views):

| App | Owner | Criticality | Strategy | Target | Wave | Dependencies | Notes |
|---|---|---|---|---|---|---|---|
| Intranet CMS | Comms | Low | Replace | SharePoint | 1 | none | Content migration |
| Orders API | Sales IT | High | Replatform | App Service + SQL MI | 3 | Orders DB, ERP | Use Copilot modernisation |
| Legacy reporting | Finance | Low | Retire | — | — | — | Replaced by Power BI |
| ERP (SAP) | Finance | Critical | Rehost | Azure VMs (SAP-certified) | 5 | many | Separate programme |

CAF provides templates (strategy, plan, readiness checklists) and the Azure Migrate business case for financial justification.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Speed vs modernisation** | Use deadlines to set the floor (rehost); fund modernisation as a follow-on with clear value. |
| **Landing zones first** | Don't migrate into unmanaged subscriptions; even a minimal landing zone avoids rework. |
| **Pilot** | Start with representative but low-risk workloads to tune the factory. |
| **Application owners** | Every workload needs a business owner for testing and sign-off. |
| **Decommissioning** | Plan and track it; savings only appear when on-premises is switched off. |
| **AI readiness** | Prioritise migrating data platforms and APIs that AI initiatives depend on. |

## Hands-on

1. Build an application inventory (from Azure Migrate or a CMDB) and assign an R to each.
2. Draft a wave plan for 20 applications using dependency information.
3. Define the factory's definition of done for a migrated workload (monitoring, backup, security, tags, documentation, decommission).
4. Write a one-page migration strategy with business outcomes and measures.
5. Identify three workloads to retire and estimate savings.

## Security, cost and reliability notes

- **Security from day one**: landing zone policies, Defender, private networking and identity must be in place before workloads land (Topic 25).
- **Cost**: parallel running during migration doubles some costs; plan timelines and commitments around it (Topic 29).
- **Reliability**: rehosted VMs inherit their old resilience; add zones, backup and DR as part of migration (Topic 28).

## Common mistakes

| Mistake | Fix |
|---|---|
| Lift-and-shift everything without optimisation | Rightsize and plan modernisation |
| Modernise everything before moving anything | Mixed strategy driven by value and deadlines |
| Ignoring dependencies | Dependency analysis and wave grouping |
| No landing zone | Ready phase first |
| Forgetting decommissioning | Track and switch off sources |

## Check yourself

1. List the Rs and when to use each.
2. What are the CAF migration phases?
3. How should waves be grouped?
4. Why should landing zones come first?
5. What's a pragmatic sequencing approach for deadline-driven migrations?
6. How do you measure migration success?

<details>
<summary>Answers</summary>

1. Retire (unused), retain (constraints), rehost (fast, minimal change), replatform (managed services with small changes), refactor (code changes for PaaS), rearchitect (major redesign), rebuild (rewrite), replace (SaaS).
2. Strategy and plan, ready, migrate, govern/manage/secure, optimise.
3. By dependencies, business priority and risk, technology pattern, downtime windows and shared data.
4. So workloads land with identity, networking, policy, monitoring and security already in place, avoiding rework and risk.
5. Rehost where deadlines bite, replatform where easy, modernise strategically afterwards.
6. Workloads migrated vs plan, cost vs business case, cutover incidents, decommissioned capacity, post-migration performance, factory velocity.

</details>

## References

- [Select your cloud migration strategies (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/plan/select-cloud-migration-strategy)
- [Plan your migration (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/migrate/plan-migration)
- [What is Azure Migrate?](https://learn.microsoft.com/azure/migrate/migrate-services-overview)
- [Azure landing zones](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
