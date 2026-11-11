# Post-Migration Optimisation

## Why it matters

Cutover day isn't the finish line. Freshly migrated workloads are usually oversized (sized from on-premises allocations or conservative assessments), not yet covered by commitments, missing some governance controls, and operated by teams who don't yet own them in Azure. Business cases assumed savings that only appear after rightsizing, commitment purchases and decommissioning on-premises infrastructure. **Post-migration optimisation** closes that gap: tune cost and performance, align with landing zone governance, harden security and resilience, hand over to operations with clear ownership, and feed a modernisation backlog. Microsoft's Cloud Adoption Framework calls this the optimise and manage phases; it's where migrations prove their value.

## Key concepts

### Optimisation areas

| Area | Actions |
|---|---|
| **Cost** | Rightsize with real Azure telemetry (2–4 weeks), schedule non-production, apply Azure Hybrid Benefit, buy reservations/savings plans for the stable baseline, remove migration leftovers (replication resources, test VMs, temporary storage) (Topic 29) |
| **Performance** | Compare against pre-migration baselines; tune disks (Premium SSD v2), VM series, SQL tiers and indexes, caching |
| **Governance alignment** | Policies compliant (tags, regions, SKUs), resources in correct management groups/subscriptions, naming, locks where needed (Topic 03) |
| **Security** | Defender recommendations, private endpoints, removal of public IPs, Entra authentication, Key Vault, patching via Azure Update Manager (Topic 25) |
| **Reliability** | Zones, backup policies, DR (Site Recovery or redeploy), health monitoring and SLOs (Topic 28) |
| **Operations** | Monitoring baselines and alerts, runbooks, on-call ownership, incident processes (Topic 26) |
| **Modernisation backlog** | Candidates for replatform/refactor based on cost, change frequency and strategic value (file 05) |
| **Decommissioning** | Switch off and remove on-premises servers, licences and contracts; update CMDB |

### Operational handover

| Item | Content |
|---|---|
| **Ownership** | Named application and technical owners in tags and CMDB |
| **Runbooks** | Start/stop, scaling, failover, restore, common incidents |
| **Monitoring** | Dashboards, alerts routed to the owning team, SLOs |
| **Access** | RBAC via groups, PIM for elevated roles, break-glass procedures |
| **Support model** | Service desk integration, escalation paths, Microsoft support plan |
| **Documentation** | Architecture diagram, dependencies, configuration, IaC repository |
| **Hypercare exit criteria** | Stable SLOs for N days, no Sev 1/2 incidents, cost within plan |

### Measuring migration value

| Metric | Source |
|---|---|
| Actual cost vs business case | Cost Management by tag |
| Rightsizing savings realised | Before/after cost |
| Commitment coverage and utilisation | Cost Management |
| Policy compliance and Secure Score | Azure Policy, Defender for Cloud |
| Availability vs SLO | Azure Monitor |
| On-premises capacity decommissioned | Migration tracker |

## How it works on Azure

30/60/90-day post-migration plan:

```text
Day 0–30 (hypercare): monitor SLOs and errors; fix performance issues; confirm backup jobs and alerts;
                      collect utilisation; remove migration artefacts; close replication
Day 30–60 (optimise): apply rightsizing (Advisor + Azure Migrate post-migration recommendations);
                      AHB everywhere eligible; schedule non-prod; fix policy non-compliance; Defender recommendations
Day 60–90 (commit and hand over): buy reservations/savings plans for stable baseline; finalise runbooks and ownership;
                      hypercare exit; decommission on-premises; prioritise modernisation backlog
```

Find migration leftovers:

```kusto
resources
| where tags["migrationTest"] == "true" or name contains "-test" or name endswith "-asr"
| project name, type, resourceGroup, subscriptionId
```

Compliance check for migrated subscriptions:

```kusto
policyresources
| where type == "microsoft.policyinsights/policystates" and properties.complianceState == "NonCompliant"
| extend policy = tostring(properties.policyDefinitionName), resourceType = tostring(properties.resourceType)
| summarize nonCompliant = count() by policy, resourceType
| order by nonCompliant desc
```

Patching and configuration for migrated VMs: Azure Update Manager maintenance configurations, Machine Configuration (guest configuration) policies, Defender for Servers.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **When to commit** | After rightsizing and usage stabilises (typically 30–90 days); cover baseline only. |
| **Hypercare length** | Proportional to criticality; define exit criteria upfront. |
| **Modernisation priority** | Rank by cost savings, operational pain, change frequency and AI/business value. |
| **Ownership model** | Application teams own workloads; platform team owns landing zone and shared services. |
| **Decommission speed** | Decommission as soon as rollback windows close to realise savings. |
| **Continuous optimisation** | Hand off to FinOps and operations cadences (Topic 29 file 07). |

## Hands-on

1. For a migrated workload, compare 30 days of Azure utilisation with assessed sizes; rightsize two VMs.
2. Apply AHB and model reservation savings for the stable baseline.
3. Run the policy compliance query and remediate the top three issues.
4. Write a hypercare exit checklist and runbook set for one application.
5. Build a modernisation backlog of five applications with scores and rationale.

## Security, cost and reliability notes

- **Migration credentials and replication appliances** should be removed or locked down after completion.
- **Public exposure introduced during migration** (temporary IPs, firewall rules) must be closed.
- **Backups and DR** must be verified with restore tests, not assumed.
- **Savings** depend on decommissioning; track it explicitly.

## Common mistakes

| Mistake | Fix |
|---|---|
| Committing to oversized VMs | Rightsize first |
| Leaving replication and test resources running | Clean-up checklist |
| No ownership after the migration team leaves | Formal handover |
| On-premises left running "just in case" | Decommission plan with dates |
| Treating migration as done | Modernisation backlog and continuous optimisation |

## Check yourself

1. List the post-migration optimisation areas.
2. What belongs in an operational handover?
3. Describe a 30/60/90-day post-migration plan.
4. When should you buy commitments after migration?
5. How do you measure migration value?
6. Why is decommissioning essential?

<details>
<summary>Answers</summary>

1. Cost, performance, governance alignment, security, reliability, operations, modernisation backlog, decommissioning.
2. Ownership, runbooks, monitoring and alerts, access model, support model, documentation, hypercare exit criteria.
3. 0–30 hypercare and data collection; 30–60 rightsizing, AHB, compliance and security fixes; 60–90 commitments, handover, decommissioning and backlog.
4. After rightsizing and when usage is stable, typically 30–90 days, covering only the baseline.
5. Cost vs business case, realised savings, commitment coverage, compliance and Secure Score, availability vs SLO, decommissioned capacity.
6. Savings only materialise when on-premises infrastructure, licences and contracts are removed.

</details>

## References

- [Azure Migrate overview](https://learn.microsoft.com/azure/migrate/migrate-services-overview)
- [CAF Migrate](https://learn.microsoft.com/azure/cloud-adoption-framework/migrate/)
- [CAF Manage](https://learn.microsoft.com/azure/cloud-adoption-framework/manage/)
- [Azure Update Manager](https://learn.microsoft.com/azure/update-manager/overview)
- [Well-Architected cost optimisation](https://learn.microsoft.com/azure/well-architected/cost-optimization/)
