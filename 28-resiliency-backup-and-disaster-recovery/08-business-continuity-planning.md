# Business Continuity Planning

## Why it matters

Technology is only part of surviving a disaster. When a region is down or ransomware has encrypted production, the questions are human: who decides to fail over, who can approve accepting data loss, who tells customers and regulators, which systems come back first, where are the runbooks and do they still work, and can people even reach them if identity is down. **Business continuity planning (BCP)** connects business impact analysis to technical recovery (disaster recovery, DR) and to people and communications. Architects and consultants are often asked to produce or review BC/DR plans; the best ones are short, owned, tested and kept current.

## Key concepts

### BCP components

| Component | Content |
|---|---|
| **Business impact analysis (BIA)** | Critical business processes, their supporting applications, impact of downtime over time (financial, regulatory, safety, reputational), resulting RTO/RPO |
| **Dependency mapping** | Applications → infrastructure → data → identity, DNS, network, third parties, people; recovery order |
| **Recovery strategies** | Per tier: active-active, warm standby, backup-restore, manual workarounds (file 02) |
| **DR plan and runbooks** | Step-by-step technical procedures, decision criteria, scripts, verification steps |
| **Roles and decision rights** | Incident commander, technical leads, business owners, who authorises failover and data-loss acceptance |
| **Communication plan** | Internal (staff, executives), external (customers, partners), regulatory notifications, status pages, templates |
| **Testing and exercise programme** | Tabletops, component tests, full DR drills, restore tests, chaos game days |
| **Maintenance** | Review after changes, incidents and at least annually |

### Disaster scenarios to plan for

| Scenario | Distinctive challenge |
|---|---|
| Zone outage | Usually handled automatically if zone-resilient |
| Regional outage | Failover decision, data loss acceptance, capacity in secondary |
| Ransomware / destructive attack | Clean recovery from immutable backups, identity compromise, forensics before restore |
| Identity outage or compromise | Break-glass access, out-of-band communications |
| Bad deployment / data corruption | Rollback and point-in-time restore rather than failover |
| Third-party/SaaS outage | Workarounds, contracts, alternate providers |
| Loss of key people or locations | Documented runbooks, cross-training |

### Decision framework

Define in advance:

- **Triggers**: for example, regional service health incident affecting critical services for more than 30 minutes with no ETA.
- **Decision makers**: named roles with deputies.
- **Data-loss authority**: who can approve forced failover with possible data loss.
- **Failback criteria**: when and how to return.

### Testing levels

| Test | Effort | Frequency |
|---|---|---|
| Plan review / walkthrough | Low | Quarterly |
| Tabletop exercise | Low–medium | Quarterly or twice yearly |
| Component test (restore a DB, fail over a SQL group) | Medium | Monthly/quarterly |
| Full DR test (test failover of the application) | High | Annually (critical: twice yearly) |
| Chaos game day | Medium | Quarterly (file 07) |

## How it works on Azure

Tooling that supports BCP:

| Need | Azure capability |
|---|---|
| Dependency discovery | Azure Resource Graph, Application Insights application map, Azure Migrate dependency analysis, Infrastructure Resiliency Manager application discovery |
| Protection overview | Azure Business Continuity Center (backup and Site Recovery estate) |
| Orchestrated recovery | Site Recovery recovery plans, IRM recovery orchestration plans (Preview), Automation runbooks, IaC pipelines |
| Platform event awareness | Service Health alerts and RCAs (Topic 26 file 08) |
| Out-of-band access | Break-glass Entra accounts (excluded from Conditional Access, monitored), offline copy of runbooks |
| Communication | Status page, Teams/email templates, Azure Communication Services for notifications |

Runbook template (excerpt):

```text
Runbook: Orders platform regional failover (West Europe → North Europe)
Trigger: Service Health incident for West Europe affecting App Service or SQL > 30 min, or SLO burn > 10x for 15 min
Decision: Incident commander + Orders business owner (deputy: Head of Ops). Data loss up to 5 s accepted (pre-approved).
Steps:
 1. Confirm scope (Service Health tracking ID, Resource Health, synthetic tests).
 2. Announce failover start (template C1) to #incident and status page.
 3. SQL failover group: az sql failover-group set-primary -g rg-orders-neu -s sql-orders-neu -n fog-orders
 4. Scale out App Service in NEU to 6 instances; verify health endpoint.
 5. Front Door: confirm origin priority switch; disable WEU origins.
 6. Verify: synthetic checkout, error rate < 1%, queue depths normal.
 7. Communicate completion (template C2); monitor for 2 hours.
Failback: after WEU healthy for 24 h and data resynced; scheduled maintenance window.
```

Store runbooks in Git (versioned), publish to a location reachable during identity or network outages, and test them in drills.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Scope** | Prioritise critical business processes; don't produce DR plans for every system at the same depth. |
| **Automation vs manual** | Automate steps; keep human decisions for failover and data loss. |
| **Plan format** | Short, actionable runbooks over long documents nobody reads. |
| **Ransomware readiness** | Clean-room recovery environment, immutable backups, identity recovery plan. |
| **Regulatory** | Align with sector rules (for example DORA for EU financial entities, NIS2) on testing and reporting. |
| **Ownership** | Each critical service has a named BC/DR owner and review date. |

## Hands-on

1. Run a lightweight BIA for three business processes; derive RTO/RPO.
2. Map dependencies for one process using Resource Graph and the application map.
3. Write a regional failover runbook using the template.
4. Run a tabletop exercise for a ransomware scenario; capture gaps.
5. Execute a component test (database failover or restore) and record achieved RTO/RPO.

## Security, cost and reliability notes

- **Break-glass access** must work when SSO/Conditional Access is impaired; test it.
- **Ransomware**: assume identity compromise; recover identity and admin access first.
- **Cost**: drills consume time and resources; budget for them as insurance.
- **Stale plans** are dangerous; tie reviews to change management.

## Common mistakes

| Mistake | Fix |
|---|---|
| DR plan written once, never tested | Regular drills |
| Undefined failover decision rights | Named roles and criteria |
| Forgetting communications | Templates and channels |
| Runbooks only in systems that may be down | Offline/alternate copies |
| Ignoring third-party dependencies | Include them in BIA and contracts |

## Check yourself

1. What does a business impact analysis produce?
2. List the components of a BC/DR plan.
3. What should be decided in advance about failover?
4. Name four disaster scenarios and their distinctive challenges.
5. Describe the levels of DR testing.
6. Which Azure tools support dependency mapping and orchestrated recovery?

<details>
<summary>Answers</summary>

1. Critical processes, supporting systems, impact over time and resulting RTO/RPO and priorities.
2. BIA, dependency mapping, recovery strategies, runbooks, roles and decision rights, communication plan, testing programme, maintenance.
3. Triggers, decision makers and deputies, data-loss authority, failback criteria.
4. Regional outage (failover decision, capacity), ransomware (clean recovery, identity compromise), identity outage (break-glass), data corruption (point-in-time restore), third-party outage (workarounds).
5. Plan reviews, tabletop exercises, component tests, full DR tests, chaos game days.
6. Resource Graph, application map, Azure Migrate dependency analysis, IRM discovery; Site Recovery recovery plans, IRM recovery orchestration plans, Automation runbooks, IaC pipelines, Business Continuity Center.

</details>

## References

- [Disaster recovery (Well-Architected)](https://learn.microsoft.com/azure/well-architected/reliability/disaster-recovery)
- [Azure Business Continuity Center](https://learn.microsoft.com/azure/business-continuity-center/business-continuity-center-overview)
- [Reliability testing strategy](https://learn.microsoft.com/azure/well-architected/reliability/testing-strategy)
- [Protect your estate (CAF Manage)](https://learn.microsoft.com/azure/cloud-adoption-framework/manage/protect)
