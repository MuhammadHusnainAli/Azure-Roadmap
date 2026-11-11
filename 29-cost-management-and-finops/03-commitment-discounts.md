# Commitment Discounts

## Why it matters

For steady workloads, paying on-demand rates is usually the most expensive way to run Azure. **Reservations** (up to 72% off pay-as-you-go for some resources), **savings plans** for compute and for databases, **Azure Hybrid Benefit** for existing licences and **Microsoft Azure Consumption Commitment (MACC)** agreements can cut costs dramatically, but each carries a commitment risk: buy too much, too specifically or for workloads that are about to change, and you pay for unused discounts. Rate optimisation is typically the single largest FinOps saving, and choosing the right instrument, scope and amount is a core architect and consultant skill.

## Key concepts

### Instruments compared

| Instrument | Commitment | Flexibility | Applies to |
|---|---|---|---|
| **Reservations** | 1 or 3 years for a specific resource type, region and size family (or per product rules) | Instance size flexibility within families (VMs); exchanges/refunds within policy limits | VMs, SQL vCores, Cosmos DB RU/s, App Service, Storage capacity, Data Explorer, Redis, Databricks units, **Foundry provisioned throughput (PTU)** and many more |
| **Savings plan for compute** | 1 or 3 years, fixed **$/hour** spend | Applies automatically across regions and eligible compute services, highest discount first | VMs, App Service, Functions Premium, Container Instances, Dedicated Host, Container Apps, Spring Apps Enterprise (infrastructure costs only) |
| **Savings plan for databases** | 1 or 3 years, fixed $/hour | Across eligible database services | Azure SQL (DB, MI, Hyperscale, serverless), PostgreSQL, MySQL, Cosmos DB, DocumentDB, DMS, SQL Server hourly licences on VMs and Arc |
| **Azure Hybrid Benefit (AHB)** | Existing licences with Software Assurance or subscriptions | Assign centrally at scope | Windows Server, SQL Server, RHEL/SLES; also AKS on Windows nodes and Arc scenarios |
| **Spot** | None (evictable) | Workload must tolerate eviction | VMs, VMSS, AKS spot node pools (file 05) |
| **MACC** | Contract to spend an amount over a term | Eligible Azure and Marketplace purchases count | Overall Azure spend; negotiated discounts and credits |

### How discounts apply

- Reservations and savings plans are **billing discounts**; they don't change resources.
- Application is **hourly**: unused commitment in an hour is lost ("use it or lose it").
- **Order of application**: reservations apply first, then savings plans, then pay-as-you-go.
- **Scope**: single resource group, single subscription, management group or shared (billing scope). Shared scope maximises utilisation.
- Savings plans apply to the usage with the highest discount percentage first.

### Choosing

| Workload | Best fit |
|---|---|
| Stable VM size/region for years (for example SAP, domain controllers) | Reservations (highest discount) |
| Changing VM families, regions, or moving to containers/App Service | Compute savings plan |
| Database estate modernising between SQL MI, Hyperscale, PostgreSQL | Database savings plan |
| Steady AI inference with predictable throughput | PTU reservations (monthly or yearly) ([Topic 15 file 05](../15-foundry-models-and-azure-openai/05-provisioned-throughput.md)) |
| Windows/SQL licences already owned | Azure Hybrid Benefit (stack with reservations/savings plans) |
| Batch, CI, dev/test tolerant of interruption | Spot |

### Recommendations and utilisation

Cost Management and Advisor provide reservation and savings plan **recommendations** based on 7, 30 or 60 days of usage, with projected savings. Track **utilisation** (target > 95%) and set utilisation alerts. Commit in **layers**: cover the steady baseline first, add incrementally as usage stabilises; avoid covering peaks.

## How it works on Azure

Typical rate optimisation process:

1. **Clean up first**: rightsize and remove waste (file 05) so you don't commit to waste.
2. **Apply AHB** wherever licences exist (Azure Policy can audit; central AHB management for SQL at scope).
3. **Reservations** for stable, specific resources.
4. **Savings plans** to cover remaining eligible compute/database spend baseline.
5. **Monitor** utilisation and coverage monthly; adjust with exchanges, new purchases or trade-ins within policy.

Query savings plan recommendations via the CLI (illustrative):

```bash
az rest --method get --url \
"https://management.azure.com/providers/Microsoft.Billing/billingAccounts/<id>/billingProfiles/<id>/providers/Microsoft.CostManagement/benefitRecommendations?api-version=2023-11-01&\$filter=properties/lookBackPeriod eq 'Last30Days' and properties/term eq 'P3Y'&\$expand=properties/allRecommendationDetails"
```

Simple break-even reasoning for a reservation: if the 3-year reserved rate is 40% of PAYG, the reservation pays off if the resource runs more than ~40% of the hours over the term; below that, PAYG or a savings plan is cheaper.

Azure Hybrid Benefit for a VM:

```bash
az vm update -g rg-app -n vm-sql01 --license-type Windows_Server
az sql vm update -g rg-app -n vm-sql01 --license-type AHUB
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Coverage target** | Commonly 70–90% of stable baseline; keep headroom for change. |
| **Term** | 3-year for very stable platforms; 1-year when change is likely. |
| **Scope** | Shared or management group scope for higher utilisation; single subscription only for strict chargeback needs. |
| **Centralised purchasing** | A FinOps team buys commitments; amortised cost allocated to consumers (file 04). |
| **Architecture changes** | Coordinate migrations and modernisation with commitment expiries. |
| **AI** | PTU reservations only after measuring stable throughput; combine with spillover to standard. |

## Hands-on

1. Review reservation and savings plan recommendations for a billing scope; compare projected savings.
2. Calculate break-even utilisation for a VM reservation using retail prices.
3. Apply Azure Hybrid Benefit to a Windows VM and a SQL VM in a test environment.
4. Create a reservation utilisation alert.
5. Build a coverage report (reserved/savings plan vs PAYG hours) from exports.

## Security, cost and reliability notes

- **Purchase permissions**: restrict who can buy commitments (Reservation Purchaser role, billing roles).
- **Compliance with licensing**: AHB requires eligible licences; track entitlements to stay compliant.
- **Commitment risk**: unused commitments are sunk cost; monitor and adjust.
- **Spot reliability**: only for interruption-tolerant workloads.

## Common mistakes

| Mistake | Fix |
|---|---|
| Buying commitments before rightsizing | Optimise usage first |
| Covering peaks, not baseline | Commit to steady baseline |
| Single-subscription scope with low utilisation | Shared/management group scope |
| Forgetting AHB | Audit and apply |
| Not monitoring utilisation | Alerts and monthly review |

## Check yourself

1. Compare reservations and savings plans.
2. Which services do compute and database savings plans cover?
3. In what order are discounts applied?
4. What is Azure Hybrid Benefit?
5. How should you size commitments?
6. When are spot VMs appropriate?

<details>
<summary>Answers</summary>

1. Reservations commit to specific resource types/regions for the highest discounts; savings plans commit to hourly spend and apply flexibly across eligible services and regions.
2. Compute: VMs, App Service, Functions Premium, Container Instances, Dedicated Host, Container Apps, Spring Apps Enterprise. Database: Azure SQL family, PostgreSQL, MySQL, Cosmos DB, DocumentDB, DMS, SQL Server hourly licences on VMs/Arc.
3. Reservations first, then savings plans, then pay-as-you-go.
4. Using existing Windows Server, SQL Server or Linux subscription licences to reduce Azure costs.
5. Clean up first, cover the steady baseline (not peaks) in layers, use shared scope, monitor utilisation.
6. For interruption-tolerant workloads such as batch, CI, dev/test and stateless scale-out.

</details>

## References

- [What are savings plans?](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview)
- [What are Azure Reservations?](https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations)
- [How reservation discounts apply](https://learn.microsoft.com/azure/cost-management-billing/reservations/reservation-discount-application)
- [Azure Hybrid Benefit for SQL](https://learn.microsoft.com/azure/azure-sql/azure-hybrid-benefit)
- [Azure consumption commitment benefit](https://learn.microsoft.com/azure/marketplace/azure-consumption-commitment-benefit)
