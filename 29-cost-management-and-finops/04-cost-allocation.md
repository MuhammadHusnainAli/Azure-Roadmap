# Cost Allocation

## Why it matters

A cloud bill that nobody owns doesn't get optimised. When costs land in one central budget, teams have no reason to switch off idle resources, rightsize or question a 10× increase in tokens. **Cost allocation** attributes every euro of spend to a team, product, environment or customer, so that owners see the consequences of their decisions (**showback**) or pay for them (**chargeback**). It depends on a deliberate **tagging strategy**, subscription and resource group design, rules for **shared costs** (networking hubs, monitoring, security tooling, commitment purchases, AI gateways) and reporting that people trust.

## Key concepts

### Allocation dimensions

| Mechanism | Strength | Weakness |
|---|---|---|
| **Billing structure** (billing profiles, invoice sections) | Separate invoices per business unit | Coarse |
| **Management groups and subscriptions** | Clean, enforceable boundaries; landing zone design | Not everything fits one subscription |
| **Resource groups** | Workload lifecycle boundary | Shared resources cross boundaries |
| **Tags** | Flexible (owner, cost centre, product, environment) | Need enforcement; not all costs support tags |
| **Cost allocation rules** | Redistribute shared costs | EA/MCA only; doesn't support purchases |
| **Telemetry-based allocation** | Usage-based splits (for example tokens per team via API Management) | Requires custom data pipeline |

### Tagging strategy

| Tag | Purpose |
|---|---|
| `owner` | Accountable person/team (group alias, not an individual where possible) |
| `costCentre` | Finance mapping |
| `workload` / `application` | Product or service |
| `environment` | prod, staging, dev, sandbox |
| `criticality` | Business tier (Topic 28) |
| `dataClassification` | Security context |
| `expires` | For sandboxes (Topic 27 file 09) |

Enforce with **Azure Policy** (require tags on resource groups, inherit tags from resource groups to resources with `modify`), and enable **tag inheritance** in Cost Management so usage records get subscription and resource group tags even when resources don't have them. Some resources and charges (for example certain marketplace or classic charges) don't carry tags; plan for an "unallocated" bucket and keep it small.

### Showback vs chargeback

| Model | Meaning | When |
|---|---|---|
| **Showback** | Report costs to owners without moving money | Early FinOps maturity; builds awareness |
| **Chargeback** | Internal billing from central budget to business units | Mature organisations with agreed allocation rules |

### Shared costs

| Shared cost | Allocation options |
|---|---|
| Hub networking, firewalls, ExpressRoute | Proportional to spoke consumption, fixed split, or as platform overhead |
| Central Log Analytics/Sentinel | By ingestion per `_ResourceId`/table (Topic 26 file 10) |
| Commitment purchases | Amortised cost to consuming resources (automatic in amortised view) |
| AI gateway and shared model deployments (PTU) | By tokens per consumer from APIM metrics |
| Support plans, security tooling | Platform overhead or headcount-based |
| AKS shared clusters | By namespace requests/usage (AKS cost analysis add-on) |

**Cost allocation rules** in Cost Management redistribute shared subscription, resource group or tag costs to targets evenly, proportionally to total/compute/storage/network cost, or by custom percentages. They affect cost analysis views, not the invoice, and require EA or MCA with appropriate billing roles.

## How it works on Azure

Policy: require `costCentre` and `owner` on resource groups and inherit them to resources (built-in definitions):

```bash
az policy assignment create --name require-rg-costcentre --scope /providers/Microsoft.Management/managementGroups/mg-landing-zones \
  --policy "96670d01-0a4d-4649-9c89-2d3abc0a5025" --params '{ "tagName": { "value": "costCentre" } }'      # Require a tag on resource groups
az policy assignment create --name inherit-costcentre --scope /providers/Microsoft.Management/managementGroups/mg-landing-zones \
  --policy "cd3aa116-8754-49c9-a813-ad46512ece54" --params '{ "tagName": { "value": "costCentre" } }' \
  --mi-system-assigned --location westeurope                                                               # Inherit a tag from the resource group
```

(Verify built-in definition IDs with `az policy definition list --query "[?displayName=='Require a tag on resource groups']"` before use.)

Tag coverage query:

```kusto
resources
| summarize total = count(), tagged = countif(isnotempty(tags.costCentre)) by subscriptionId
| extend coveragePct = round(100.0 * tagged / total, 1)
```

Token-based chargeback for a shared AI gateway: APIM emits token metrics with a `consumer` dimension; a monthly job multiplies each consumer's tokens by the effective model price (including PTU amortisation) and publishes the result alongside Cost Management exports in Power BI or FinOps hubs.

Showback report: monthly amortised cost by `costCentre` and `workload` with trend, unallocated percentage, top changes and optimisation opportunities, sent to owners.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Subscription-first design** | Align subscriptions with workloads/environments for clean allocation; tags refine. |
| **Minimal tag set** | 4–6 mandatory tags; more becomes unenforceable. |
| **Enforce at resource group** | Require on resource groups, inherit to resources. |
| **Shared cost policy** | Agree rules with finance up front; keep them simple and stable. |
| **Showback before chargeback** | Build trust in data first. |
| **Unallocated target** | Keep below ~5%; investigate the remainder monthly. |

## Hands-on

1. Define a tag taxonomy with owners; assign require-and-inherit policies to a test management group.
2. Enable tag inheritance in Cost Management; compare cost by tag before and after.
3. Create a cost allocation rule distributing a hub subscription's cost to spokes proportionally.
4. Run the tag coverage query and fix gaps.
5. Build a showback Power BI report from exports grouped by `costCentre`.

## Security, cost and reliability notes

- **Tags aren't security boundaries**: don't use them for access control decisions.
- **Data quality**: wrong tags produce wrong bills; validate with policy and audits.
- **Sensitive data**: avoid putting personal data in tags.

## Common mistakes

| Mistake | Fix |
|---|---|
| Optional tags | Policy enforcement and inheritance |
| Dozens of tags | Minimal mandatory set |
| Shared costs left in one central bucket | Agreed allocation rules |
| Chargeback before data is trusted | Showback first |
| Allocating actual cost with lumpy reservation purchases | Amortised cost |

## Check yourself

1. List the mechanisms for allocating costs.
2. Which tags would you mandate?
3. How do you enforce tagging?
4. Compare showback and chargeback.
5. What do cost allocation rules do, and what don't they affect?
6. How can you allocate shared AI gateway costs?

<details>
<summary>Answers</summary>

1. Billing structure, management groups/subscriptions, resource groups, tags, cost allocation rules, telemetry-based allocation.
2. owner, costCentre, workload, environment, criticality (plus dataClassification or expires as needed).
3. Azure Policy to require tags on resource groups and inherit to resources, plus Cost Management tag inheritance.
4. Showback reports costs to owners; chargeback bills them internally.
5. Redistribute shared costs in cost analysis views by even, proportional or custom splits; they don't change the invoice or apply to purchases.
6. By tokens per consumer from API Management metrics multiplied by effective model prices.

</details>

## References

- [Cost allocation rules](https://learn.microsoft.com/azure/cost-management-billing/costs/allocate-costs)
- [Introduction to cost allocation](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-allocation-introduction)
- [Tag inheritance](https://learn.microsoft.com/azure/cost-management-billing/costs/enable-tag-inheritance)
- [Use tags to organise resources](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources)
