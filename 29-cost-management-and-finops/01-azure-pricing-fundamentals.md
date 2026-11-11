# Azure Pricing Fundamentals

## Why it matters

Every Azure design decision is also a pricing decision: SKU, region, redundancy, scaling model, data movement, licensing. Customers judge architects and consultants on whether the bill matches the estimate, and the gap is usually caused by misunderstood meters (egress, transactions, log ingestion, tokens), the wrong billing scope, or forgotten services. Understanding how Azure prices resources, how billing accounts and scopes are organised, how to read an invoice and how to use the pricing calculator is the foundation for cost management (file 02), estimation (file 08) and every FinOps practice in this topic.

## Key concepts

### Pricing models

| Model | Description | Examples |
|---|---|---|
| **Pay-as-you-go** (consumption) | Pay per unit used, per hour/second/GB/transaction/token | VMs per hour, storage per GB-month, Functions per execution, tokens |
| **Provisioned capacity** | Pay for reserved capacity whether used or not | PTUs, Cosmos DB provisioned RU/s, dedicated SQL vCores, Premium SKUs |
| **Commitment discounts** | Lower rates in exchange for 1- or 3-year commitments | Reservations, savings plans (file 03) |
| **Licensing benefits** | Bring existing licences | Azure Hybrid Benefit for Windows Server, SQL Server, RHEL/SLES |
| **Free tiers and grants** | Monthly free amounts or free services | Functions free grant, free accounts, some free tables in Log Analytics |
| **Spot** | Deep discounts on evictable capacity | Spot VMs (file 05) |
| **Marketplace** | Third-party software billed through Azure | Can count towards MACC if eligible |

### Meters

Every chargeable service emits usage against **meters** (for example "D4s v5 – West Europe – Linux – per hour", "LRS Data Stored – per GB/month", "Data Transfer Out – Zone 1 – per GB"). Prices vary by **region**, **SKU/tier**, **OS/licence**, **redundancy** and sometimes **time** (reservation vs PAYG). Common surprises:

| Hidden driver | Note |
|---|---|
| **Data transfer** | Internet egress, inter-region transfer, cross-zone traffic for some services, Private Link data processing, VNet peering |
| **Transactions/operations** | Storage operations, Key Vault operations, API calls |
| **Log ingestion** | Log Analytics/Sentinel per GB (Topic 26 file 10) |
| **Idle provisioned capacity** | Unused PTUs, over-provisioned RU/s, stopped-but-not-deallocated VMs, unattached disks, public IPs |
| **Tokens** | Input, cached input and output tokens priced differently per model (file 06) |
| **Backup and snapshots** | Protected instances plus storage |
| **Support plans** | Billed separately |

### Billing accounts and scopes

| Agreement | Structure |
|---|---|
| **Microsoft Customer Agreement (MCA)** | Billing account → **billing profiles** (each gets an invoice) → **invoice sections** → subscriptions |
| **Enterprise Agreement (EA)** | Enrolment → departments → accounts → subscriptions (being superseded by MCA for many customers) |
| **CSP** | Partner-managed billing |
| **Pay-as-you-go (MOSP)** | Individual subscriptions with credit card or invoice |

Resource scopes for cost analysis: management group → subscription → resource group → resource. Billing scopes (billing account, billing profile, invoice section) show invoiced cost, credits and purchases. Know which scope you're looking at: commitment purchases appear at billing scopes and are allocated to resources as amortised cost.

### Actual vs amortised cost

| View | Shows |
|---|---|
| **Actual cost** | Charges as billed, including upfront reservation purchases in the month bought |
| **Amortised cost** | Reservation/savings plan costs spread over the term and allocated to resources that used the benefit; better for showback |

### Pricing tools

| Tool | Use |
|---|---|
| **Pricing calculator** | Build estimates with regions, SKUs, discounts; export/share |
| **Azure Retail Prices API** | Programmatic, unauthenticated list prices for automation |
| **TCO calculator** | Compare on-premises vs Azure (file 08) |
| **Azure Migrate business case** | Estimate migration costs from discovered data (Topic 30) |
| **Cost Management** | Actual and forecast costs (file 02) |

## How it works on Azure

Query list prices with the Retail Prices API:

```bash
curl -s "https://prices.azure.com/api/retail/prices?\$filter=serviceName eq 'Virtual Machines' and armRegionName eq 'westeurope' and armSkuName eq 'Standard_D4s_v5' and priceType eq 'Consumption'" \
  | jq '.Items[] | {productName, skuName, retailPrice, unitOfMeasure, type}'
```

Reservation price for comparison: change `priceType eq 'Reservation'` and inspect `reservationTerm`.

Understanding an invoice line: billing profile invoice → invoice section → subscription → service family → meter → quantity × unit price (minus discounts and credits). Download usage details (cost details) or use exports for full granularity (file 02).

Find costly "invisible" resources:

```kusto
resources
| where type =~ "microsoft.compute/disks" and properties.diskState == "Unattached"
| project name, resourceGroup, sku = tostring(sku.name), sizeGB = toint(properties.diskSizeGB)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Region** | Price differences of 10–30% exist; balance with latency, residency and service availability. |
| **Consumption vs provisioned** | Consumption for variable or low usage; provisioned for steady high utilisation with predictability needs. |
| **Data movement** | Co-locate compute and data; minimise cross-region and internet egress; use CDN/Front Door caching. |
| **Licensing** | Apply Azure Hybrid Benefit wherever licences allow. |
| **Billing structure** | Billing profiles/invoice sections aligned to business units to simplify chargeback. |
| **Price awareness in design reviews** | Include a cost estimate with every architecture decision record. |

## Hands-on

1. Price a three-tier workload in the pricing calculator for two regions; compare.
2. Use the Retail Prices API to fetch PAYG and 3-year reservation prices for a VM SKU.
3. Open Cost Management at a billing profile and at a subscription; compare actual vs amortised views.
4. Run the unattached disk query and estimate monthly waste.
5. Identify data transfer meters in last month's cost details.

## Security, cost and reliability notes

- **Billing access** is sensitive; use Cost Management Reader/Billing reader roles rather than Owner.
- **Reliability costs money**: zone redundancy and multi-region change estimates; model them explicitly (Topic 28).
- **Prices change**: refresh estimates before proposals; quote assumptions and dates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Ignoring egress and transactions in estimates | Model data flows |
| Comparing actual cost months with reservation purchases | Use amortised cost |
| Stopped (not deallocated) VMs | Deallocate or delete |
| Forgetting monitoring, backup and support costs | Include supporting services |
| Using list prices for EA/MCA customers | Apply negotiated rates |

## Check yourself

1. Name five pricing models on Azure.
2. What is a meter, and what affects its price?
3. List five hidden cost drivers.
4. Describe the MCA billing hierarchy.
5. When do you use amortised rather than actual cost?
6. Which tools help estimate and analyse prices?

<details>
<summary>Answers</summary>

1. Pay-as-you-go, provisioned capacity, commitment discounts, licensing benefits, spot (also free tiers and marketplace).
2. A unit of billable usage; price varies with region, SKU/tier, OS/licence, redundancy and pricing type.
3. Any five: data transfer, transactions, log ingestion, idle provisioned capacity, tokens, backups/snapshots, support plans.
4. Billing account → billing profiles (invoices) → invoice sections → subscriptions.
5. For showback and trend analysis, because commitment costs are spread over the term and allocated to consuming resources.
6. Pricing calculator, Retail Prices API, TCO calculator, Azure Migrate business case, Cost Management.

</details>

## References

- [Microsoft Cost Management overview](https://learn.microsoft.com/azure/cost-management-billing/costs/overview-cost-management)
- [Microsoft Customer Agreement overview](https://learn.microsoft.com/azure/cost-management-billing/understand/mca-overview)
- [Analyse unexpected charges](https://learn.microsoft.com/azure/cost-management-billing/understand/analyze-unexpected-charges)
- [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)
- [Azure Retail Prices API](https://learn.microsoft.com/rest/api/cost-management/retail-prices/azure-retail-prices)
