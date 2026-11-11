# Multitenant Architecture

## Why it matters

Software-as-a-service (SaaS) providers, ISVs and internal platform teams serve many customers or business units (**tenants**) from shared infrastructure. Multitenancy determines unit economics (how cheaply you can onboard tenant 1,001), isolation (whether one tenant can see or slow down another), compliance (whether a regulated tenant can have dedicated resources) and operations (how you deploy and upgrade thousands of tenants). The Azure Architecture Center's multitenant guidance and the Well-Architected SaaS workload guidance describe tenancy models, isolation options per service, the noisy neighbour problem and per-tenant cost measurement. Multitenant AI adds new isolation concerns covered in [Topic 34 file 08](../34-ai-solution-architecture/08-multitenant-ai-solutions.md).

## Key concepts

### Tenancy models

| Model | Description | Isolation | Cost per tenant | Operations |
|---|---|---|---|---|
| **Fully multitenant (shared everything)** | All tenants share app and data stores (tenant ID in rows) | Logical | Lowest | Simplest per tenant; careful code needed |
| **Vertically partitioned** | Some tenants (premium/regulated) on dedicated resources, others shared | Mixed | Mixed | Moderate |
| **Horizontally partitioned** | Shared app tier, separate data stores per tenant (database-per-tenant) | Data isolated | Medium | More databases to manage |
| **Single tenant (stamp per tenant)** | Dedicated deployment per tenant | Strongest | Highest | Many deployments; automation essential |
| **Deployment stamps** | Groups of tenants per stamp (scale unit) | Per stamp | Balanced | Standard pattern at scale |

### Isolation dimensions

| Dimension | Options |
|---|---|
| **Identity** | Entra multitenant apps, Entra External ID for customer identities, per-tenant roles, tenant claims in tokens |
| **Compute** | Shared app with tenant context; per-tenant containers/namespaces; dedicated instances |
| **Data** | Row-level (tenant ID + row-level security), schema-per-tenant, database-per-tenant (elastic pools), account-per-tenant (Cosmos DB, Storage) |
| **Network** | Shared endpoints vs per-tenant private endpoints/custom domains |
| **Encryption** | Per-tenant keys (CMK) for premium/regulated tenants |
| **Performance** | Rate limits, quotas, resource governance per tenant |

### Noisy neighbour

One tenant's heavy usage degrades others. Mitigations: per-tenant throttling (APIM), resource governance (Cosmos DB partitioning and throughput, SQL elastic pool limits), queue-based load levelling, bulkheads (separate pools for heavy tenants), moving large tenants to dedicated stamps, and monitoring per-tenant consumption.

### Per-tenant cost and metering

Measure consumption per tenant (requests, storage, compute time, tokens) via telemetry with tenant IDs, resource tagging for dedicated resources, and allocation for shared resources. Use it for pricing tiers, profitability and capacity planning.

### Lifecycle

Tenant onboarding (automated provisioning, DNS, identity, data), configuration per tenant, upgrades (rings by tenant), offboarding (data export and deletion), and moving tenants between stamps.

## How it works on Azure

Typical SaaS architecture with stamps:

```text
Global: Front Door (custom domains per tenant) → tenant routing service/catalogue (Cosmos DB) → stamp selection
Stamp N (region): APIM (per-tenant rate limits) → Container Apps/AKS (shared app, tenant context from token)
                  → Azure SQL elastic pool (database per tenant) or Cosmos DB (partition key = tenantId)
                  → Storage (container per tenant) → AI Search (index per tenant or filter) → Key Vault (per-tenant keys for premium)
Control plane: onboarding automation (Durable Functions/pipelines), metering (Event Hubs → Fabric), billing integration
```

Row-level security in Azure SQL for shared tables:

```sql
CREATE FUNCTION security.fn_tenantAccess(@TenantId int)
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS allowed WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS int);

CREATE SECURITY POLICY security.TenantPolicy
ADD FILTER PREDICATE security.fn_tenantAccess(TenantId) ON dbo.Orders,
ADD BLOCK PREDICATE security.fn_tenantAccess(TenantId) ON dbo.Orders
WITH (STATE = ON);
-- The app sets SESSION_CONTEXT('TenantId') from the validated token on each connection.
```

Per-tenant metering query (Application Insights custom dimension `tenantId`):

```kusto
AppRequests
| where TimeGenerated > startofmonth(now())
| summarize requests = count(), cpuMs = sum(DurationMs) by tenantId = tostring(Properties.tenantId)
| order by requests desc
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Default model** | Shared app, partitioned data, with stamps for scale; dedicated options for premium/regulated tenants. |
| **Data isolation** | Database-per-tenant for strong isolation and easy offboarding; shared with RLS for many small tenants. |
| **Tenant context** | Derive from authenticated tokens, never from client-supplied parameters. |
| **Limits** | Know service limits (databases per pool, indexes per search service) when choosing per-tenant resources. |
| **Automation** | Fully automated onboarding and upgrades; manual steps don't scale. |
| **Pricing alignment** | Tenancy model should support your pricing tiers and margins. |

## Hands-on

1. Choose a tenancy model for a B2B SaaS with 2,000 small tenants and 10 regulated enterprise tenants.
2. Implement RLS in Azure SQL with session context from a token.
3. Configure APIM per-tenant rate limits keyed on a tenant claim.
4. Add tenant IDs to telemetry and build a per-tenant consumption report.
5. Design onboarding automation and a tenant move between stamps.

## Security, cost and reliability notes

- **Cross-tenant data leakage** is the top risk: test isolation explicitly (automated tests, penetration tests).
- **Per-tenant keys** for tenants requiring crypto isolation.
- **Cost**: dedicated resources for every tenant destroy margins; shared resources require governance.
- **Blast radius**: stamps limit the impact of failures and bad deployments.

## Common mistakes

| Mistake | Fix |
|---|---|
| Tenant ID from request parameters | From validated tokens |
| One huge shared database for everything | Partitioning and stamps |
| No per-tenant throttling | APIM and resource governance |
| Manual onboarding | Automation |
| No per-tenant cost data | Metering |

## Check yourself

1. Describe four tenancy models.
2. List the isolation dimensions.
3. What is the noisy neighbour problem, and how do you mitigate it?
4. How do you measure per-tenant cost?
5. What are deployment stamps for in SaaS?
6. How should tenant context be established?

<details>
<summary>Answers</summary>

1. Fully multitenant (shared everything), vertically partitioned (mixed dedicated/shared), horizontally partitioned (shared app, separate data), single-tenant stamps (dedicated per tenant); plus deployment stamps grouping tenants.
2. Identity, compute, data, network, encryption, performance.
3. One tenant's load degrades others; throttling, resource governance, load levelling, bulkheads, moving heavy tenants, monitoring.
4. Telemetry with tenant IDs, tags for dedicated resources, allocation of shared costs.
5. Scale units hosting groups of tenants, limiting blast radius and enabling regional placement.
6. From validated authentication tokens/claims, never from client-supplied parameters.

</details>

## References

- [Multitenant architecture overview](https://learn.microsoft.com/azure/architecture/guide/multitenant/overview)
- [Tenancy models](https://learn.microsoft.com/azure/architecture/guide/multitenant/considerations/tenancy-models)
- [Noisy neighbour antipattern](https://learn.microsoft.com/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor)
- [Measure consumption per tenant](https://learn.microsoft.com/azure/architecture/guide/multitenant/considerations/measure-consumption)
- [Well-Architected SaaS workloads](https://learn.microsoft.com/azure/well-architected/saas/)
