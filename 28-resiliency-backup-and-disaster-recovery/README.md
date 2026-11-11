# 28 · Resiliency, Backup and Disaster Recovery

> Tracks: E ○ A ● C ○ · Part E · Secure, Operate, Migrate and Extend

Things fail: disks, datacentres, regions, deployments, people and security controls. This topic covers designing workloads to keep running and to recover: reliability targets (SLOs, RTO, RPO) and failure mode analysis; availability zones, multi-region patterns and region selection; Azure Backup with secure-by-default soft delete, immutable vaults and multi-user authorisation; Azure Site Recovery with recovery plans and test failovers; resilience options for Storage, databases, messaging, search and Key Vault; resilience for AI workloads, including the fact that Foundry doesn't provide automatic failover; chaos engineering with Chaos Studio and Azure Infrastructure Resiliency Manager; and business continuity planning with runbooks, drills and communications.

## What you will be able to do

- Agree SLOs, RTO and RPO per user flow and run failure mode analysis.
- Make workloads zone-resilient and choose an appropriate multi-region pattern.
- Protect data with isolated, immutable, tested backups.
- Replicate and fail over VMs with Site Recovery recovery plans.
- Design data-tier DR with the right replication and failover options.
- Make AI model access, knowledge and agent state recoverable.
- Validate resilience with chaos experiments and zone-down drills.
- Produce and exercise a business continuity plan.

## Prerequisites

[Topic 05](../05-networking/README.md) (global routing), [Topic 08](../08-storage/README.md) and [Topic 09](../09-databases/README.md) (data services), [Topic 26](../26-monitoring-and-observability/README.md) (SLIs and alerts).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Reliability Fundamentals](01-reliability-fundamentals.md) | 35 min | SLOs, composite SLA, FMA |
| 02 | [Availability Zones and Multi-Region Design](02-availability-zones-and-multi-region-design.md) | 45 min | Zone-redundant deploy, active-passive |
| 03 | [Azure Backup](03-azure-backup.md) | 45 min | Immutable vault, MUA, restore test |
| 04 | [Azure Site Recovery](04-azure-site-recovery.md) | 40 min | Recovery plan, test failover |
| 05 | [Data Tier Resilience](05-data-tier-resilience.md) | 45 min | Storage failover, SQL failover group |
| 06 | [Resilience for AI Workloads](06-resilience-for-ai-workloads.md) | 40 min | APIM backend pool failover |
| 07 | [Chaos Engineering and Resiliency Tooling](07-chaos-engineering-and-resiliency-tooling.md) | 40 min | Zone-down scenario, IRM goals |
| 08 | [Business Continuity Planning](08-business-continuity-planning.md) | 35 min | BIA, runbook, tabletop |

**Architects:** all of it. **AI engineers:** 01, 02 and 06. **Consultants:** 01, 02 and 08.

## Completion checklist

- [ ] I can define SLOs, RTO and RPO and calculate composite availability.
- [ ] I have made a workload zone-resilient and designed a multi-region pattern.
- [ ] I have configured immutable, MUA-protected backups and tested a restore.
- [ ] I have run a Site Recovery test failover with a recovery plan.
- [ ] I have tested Storage and SQL failovers and know their data-loss implications.
- [ ] I have a resilient model access path and recoverable agent state.
- [ ] I have run a chaos scenario under load and a zone-down drill.
- [ ] I have written and exercised a failover runbook.

## Maps to

- **AZ-305**: design business continuity (backup, DR, high availability).
- **AZ-104**: implement backup and recovery.
- **AI-103** (adjacent): resilient AI solutions.

## Next

29 · [Cost Management and FinOps](../29-cost-management-and-finops/README.md)
