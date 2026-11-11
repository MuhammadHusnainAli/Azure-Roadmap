# 30 · Migration and Modernisation

> Tracks: E – A ● C ● · Part E · Secure, Operate, Migrate and Extend

Most Azure programmes include moving existing workloads and making them better once they're there. This topic covers migration strategy with the Rs and wave planning; Azure Migrate for discovery, agentless dependency analysis, assessments, business cases and the Azure Copilot migration agent; server migration to Azure VMs and Azure VMware Solution; database migration paths for SQL Server, Oracle, PostgreSQL, MySQL and MongoDB; application modernisation to App Service, Container Apps and AKS with GitHub Copilot modernization; data and analytics migration to Microsoft Fabric and Azure Databricks; migrating AI workloads from hubs, the Assistants API, classic agents, Semantic Kernel, AutoGen, prompt flow and other clouds to current Foundry; and post-migration optimisation and handover.

## What you will be able to do

- Choose a migration strategy per workload and plan waves around dependencies.
- Use Azure Migrate to discover, assess and build a business case.
- Migrate servers to Azure VMs or Azure VMware Solution with tested cutovers.
- Plan database migrations with the right targets, tools and rollback points.
- Modernise applications using PaaS targets and AI-assisted upgrade tooling.
- Migrate warehouses and pipelines to Fabric or Databricks with validated parity.
- Move AI workloads to current Foundry capabilities ahead of retirements.
- Optimise, govern and hand over migrated workloads to realise the business case.

## Prerequisites

[Topic 03](../03-governance-and-landing-zones/README.md) (landing zones), [Topic 06](../06-compute/README.md), [Topic 09](../09-databases/README.md), [Topic 10](../10-application-platform/README.md) and [Topic 12](../12-data-and-analytics/README.md) for target services; [Topic 29](../29-cost-management-and-finops/README.md) for business cases.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Migration Strategy](01-migration-strategy.md) | 35 min | Rationalisation and wave plan |
| 02 | [Azure Migrate](02-azure-migrate.md) | 45 min | Discovery, assessments, business case |
| 03 | [Server and VMware Migration](03-server-and-vmware-migration.md) | 45 min | Test migration and cutover |
| 04 | [Database Migration](04-database-migration.md) | 40 min | Online migration to SQL MI |
| 05 | [Application Modernisation](05-application-modernisation.md) | 45 min | Copilot modernization, Container Apps |
| 06 | [Data and Analytics Migration](06-data-and-analytics-migration.md) | 40 min | Fabric Migration Assistant, validation |
| 07 | [Migrating AI Workloads to Foundry](07-migrating-ai-workloads-to-foundry.md) | 40 min | Assistants API to new agents |
| 08 | [Post-Migration Optimisation](08-post-migration-optimisation.md) | 35 min | 30/60/90 plan, handover |

**Architects:** all of it. **Consultants:** 01, 02, 05, 07 and 08. **AI engineers** (optional track): 05 and 07.

## Completion checklist

- [ ] I have rationalised an application portfolio with the Rs and built a wave plan.
- [ ] I have run Azure Migrate discovery, dependency analysis, assessments and a business case.
- [ ] I have completed a test migration and cutover with rollback steps.
- [ ] I can choose database targets and plan online cutovers with a point of no return.
- [ ] I have modernised an application with Copilot modernization and deployed it to PaaS.
- [ ] I have migrated a warehouse or pipeline to Fabric or Databricks and validated parity.
- [ ] I have migrated an Assistants API or classic agent app to new Foundry agents.
- [ ] I have executed a post-migration optimisation and handover plan.

## Maps to

- **AZ-305**: design migrations.
- **AZ-104** (adjacent): migrate servers and manage VMs.
- **AI-103** (adjacent): migrating AI apps and agents to Foundry.

## Next

31 · [Hybrid, Multicloud and Edge](../31-hybrid-multicloud-and-edge/README.md)
