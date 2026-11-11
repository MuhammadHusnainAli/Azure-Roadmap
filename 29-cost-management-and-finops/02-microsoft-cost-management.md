# Microsoft Cost Management

## Why it matters

Cloud costs change daily as teams deploy, scale and experiment. Without continuous visibility, organisations find out about overspend when the invoice arrives, weeks after the money was spent. **Microsoft Cost Management** is the built-in toolset for analysing, monitoring and optimising Azure (and Microsoft Cloud) costs: **cost analysis** to explore spend by any dimension, **budgets** and **alerts** to act before limits are exceeded, **anomaly detection** for unexpected spikes, **exports** to feed FinOps data platforms, and integration with Advisor recommendations. It's free for Azure usage and is the "Inform" engine of the FinOps practice (file 07).

## Key concepts

### Capabilities

| Capability | What it does |
|---|---|
| **Cost analysis** | Explore actual or amortised cost by service, resource, resource group, tag, location, meter, pricing model; smart views (resources, services, reservations, subscriptions) and customisable views; forecasts |
| **Budgets** | Cost or forecast thresholds at any scope with alert recipients and action groups |
| **Anomaly alerts** | Daily detection of unexpected spikes or dips in normalised usage (subscription scope) |
| **Scheduled alerts** | Regular emailed views |
| **Reservation/savings plan utilisation alerts** | Notify when commitments are underused |
| **Exports** | Scheduled export of cost details (including the **FOCUS** format) to Storage for analysis in Power BI, Fabric or FinOps hubs |
| **Cost Details API / Query API** | Programmatic access |
| **Cost allocation rules** | Redistribute shared costs (file 04) |
| **Tag inheritance** | Apply subscription/resource group tags to usage records |
| **Recommendations** | Advisor cost recommendations surfaced in Cost Management |
| **Copilot assistance** | Natural-language cost questions in Azure Copilot (Topic 24 file 08) |

### Scopes and roles

| Role | Access |
|---|---|
| **Cost Management Reader** | View costs, budgets, exports at resource scopes |
| **Cost Management Contributor** | Manage budgets, exports, views |
| **Billing reader / billing roles** | Billing scopes (MCA/EA) |

Use management group scope for an organisational view and billing profile/invoice section scopes for invoiced amounts.

### Datasets

| Dataset | Notes |
|---|---|
| **Actual cost** | As billed |
| **Amortised cost** | Commitments spread and allocated to usage |
| **FOCUS** | FinOps Open Cost and Usage Specification, a vendor-neutral schema combining actual and amortised views; recommended for multi-cloud FinOps |

Data latency: usage typically appears within hours, but allow up to 24–48 hours for completeness; finalised after month-end.

## How it works on Azure

Budget with action group at subscription scope (Bicep):

```bicep
targetScope = 'subscription'

resource budget 'Microsoft.Consumption/budgets@2023-11-01' = {
  name: 'budget-orders-prod'
  properties: {
    category: 'Cost'
    amount: 20000
    timeGrain: 'Monthly'
    timePeriod: { startDate: '2026-12-01' }
    filter: { tags: { name: 'workload', operator: 'In', values: [ 'orders' ] } }
    notifications: {
      actual80: { enabled: true, operator: 'GreaterThan', threshold: 80, thresholdType: 'Actual', contactEmails: [ 'orders-owners@contoso.com' ] }
      forecast100: { enabled: true, operator: 'GreaterThan', threshold: 100, thresholdType: 'Forecasted',
                     contactGroups: [ actionGroupId ], contactEmails: [ 'finops@contoso.com' ] }
    }
  }
}
```

Action groups can trigger automation (for example a Logic App that notifies Teams or scales down non-production resources). Budgets don't stop resources by themselves.

Daily FOCUS export to Storage:

```bash
az costmanagement export create --name daily-focus --scope "/subscriptions/<sub>" \
  --storage-account-id /subscriptions/<sub>/resourceGroups/rg-finops/providers/Microsoft.Storage/storageAccounts/stfinops \
  --storage-container exports --storage-directory focus \
  --timeframe MonthToDate --recurrence Daily --recurrence-period from="2026-12-01T00:00:00" to="2027-12-01T00:00:00" \
  --schedule-status Active --type FocusCost
```

(Export types and CLI parameters evolve; the portal's Exports page lists current dataset versions including FOCUS.)

Investigating a spike (cost analysis): set the view to daily granularity, group by service then by resource, filter the spike day, switch to meter to find the driver (for example "Data Transfer Out"), then check Activity Log and resource metrics for the cause.

Anomaly alerts: enable per subscription and route to owners; they flag unusual daily changes without you defining thresholds.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Budget granularity** | Budgets per subscription/workload with owners; organisational budgets at management group or billing scope. |
| **Forecast alerts** | Use forecast thresholds to act before overspend, not just actuals. |
| **Data platform** | Cost Management views for day-to-day; exports to FinOps hubs/Fabric/Power BI for chargeback, multi-cloud and long-term analysis. |
| **FOCUS** | Adopt FOCUS exports for consistent schemas across clouds and tools. |
| **Automation** | Automate non-production shutdown on budget breaches; never automate deletion of production. |
| **Access** | Give engineering teams cost visibility for their scopes. |

## Hands-on

1. Build a cost analysis view grouped by tag `workload` and service for the last 3 months; save and share it.
2. Create a monthly budget with actual and forecast alerts wired to an action group.
3. Enable anomaly alerts for a subscription.
4. Configure a daily FOCUS export to Storage and load it into Power BI or Fabric.
5. Investigate the largest daily cost increase in the last 30 days down to the meter.

## Security, cost and reliability notes

- **Least privilege** for cost data; billing data can reveal commercial terms.
- **Budgets don't cap spend** (except spending limits on some offer types); combine with policy and automation.
- **Data latency**: don't use cost data for real-time protection; use quotas, APIM token limits and Azure Monitor for that.

## Common mistakes

| Mistake | Fix |
|---|---|
| Waiting for the invoice | Daily views, budgets, anomaly alerts |
| Budgets with no owner or action | Route to owners with action groups |
| Analysing actual cost after buying reservations | Amortised view |
| Each team building its own spreadsheets | Exports to a shared FinOps data platform |
| Assuming budgets stop spending | Automate responses or use quotas |

## Check yourself

1. List six Cost Management capabilities.
2. Compare actual, amortised and FOCUS datasets.
3. What can a budget alert trigger?
4. What do anomaly alerts detect?
5. How do you feed costs into a FinOps data platform?
6. Which roles provide cost visibility without broader access?

<details>
<summary>Answers</summary>

1. Any six: cost analysis, budgets, anomaly alerts, scheduled alerts, commitment utilisation alerts, exports, APIs, cost allocation, tag inheritance, recommendations.
2. Actual: as billed; amortised: commitments spread over usage; FOCUS: vendor-neutral schema combining both for multi-cloud FinOps.
3. Emails and action groups (Logic Apps, Functions, webhooks, notifications).
4. Unexpected daily spikes or dips in normalised usage at subscription scope.
5. Scheduled exports (for example FOCUS) to Storage, consumed by FinOps hubs, Fabric or Power BI.
6. Cost Management Reader (and Billing reader at billing scopes).

</details>

## References

- [Cost Management overview](https://learn.microsoft.com/azure/cost-management-billing/costs/overview-cost-management)
- [Create and manage budgets](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [Exports](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-improved-exports)
- [Common cost analysis uses](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-analysis-common-uses)
- [Cost Management best practices](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices)
