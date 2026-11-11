# Observability Cost Control

## Why it matters

Observability bills grow quietly. A debug log level left on, a chatty container, verbose firewall logs, 100% trace sampling on a high-traffic API or a Sentinel workspace ingesting everything can make monitoring one of the largest line items in an Azure invoice, sometimes more than the workloads it watches. The opposite failure is cutting so aggressively that incidents can't be investigated. Cost control means deciding **what data has what value**, then using Azure Monitor's levers (collection filters, ingestion-time transformations, sampling, table plans, retention tiers, commitment tiers and summary rules) to pay the right price for each kind of data.

## Key concepts

### Where the money goes

| Cost | Driver |
|---|---|
| **Log ingestion** | GB ingested per table plan (Analytics > Basic > Auxiliary); usually the largest cost |
| **Retention** | Interactive retention beyond included days; long-term retention per GB/month |
| **Queries** | Basic and Auxiliary queries billed per GB scanned; search jobs |
| **Metrics** | Custom metrics and Prometheus samples ingested and queried (platform metrics free) |
| **Alerts** | Log search alerts per rule frequency; metric alerts per time series |
| **Features** | Application Insights, Sentinel (on top of Log Analytics), traffic analytics, data export |

Some tables are free (for example `AzureActivity`, `Heartbeat`, `Usage`, `Operation`); the `_IsBillable` and `_BilledSize` columns show what you pay for.

### Levers

| Lever | Effect |
|---|---|
| **Collect less at source** | Disable unneeded diagnostic categories; set app log levels; XPath filters for events; namespace filters for containers |
| **DCR transformations** | KQL at ingestion to drop rows/columns, mask data, or split data into cheaper tables |
| **Sampling** | Application Insights/OpenTelemetry sampling keeps representative traces |
| **Table plans** | Analytics for alerting/frequent queries; **Basic** for troubleshooting; **Auxiliary/Lake** for verbose, rarely queried data (file 02) |
| **Retention** | Keep interactive retention short; long-term retention (up to 12 years) for compliance |
| **Summary rules** | Aggregate verbose data into small Analytics tables for dashboards and alerts |
| **Commitment tiers** | From 100 GB/day; save up to ~30% versus pay-as-you-go for **Analytics** ingestion (31-day commitment; Basic/Auxiliary not covered) |
| **Dedicated clusters** | Pool commitment across workspaces; CMK and other features |
| **Daily cap** | Safety net to stop runaway ingestion (data beyond the cap is lost; use for non-critical workspaces) |
| **Sentinel data lake** | Low-cost long-term security data (Topic 25 file 06) |

### Data value tiers

| Value | Examples | Treatment |
|---|---|---|
| **Detect and alert** | Error metrics, Entra sign-ins, security alerts, Key Vault audit | Analytics; retention for detection windows |
| **Troubleshoot** | Application debug logs, container stdout, detailed diagnostics | Basic; short interactive retention |
| **Comply and forensics** | Verbose firewall/flow logs, raw proxy logs, audit for regulators | Auxiliary/lake; long total retention |
| **Not needed** | Duplicate logs, health probes, noisy informational events | Drop at source or with transformations |

## How it works on Azure

Find the biggest tables:

```kusto
Usage
| where TimeGenerated > ago(30d) and IsBillable
| summarize GB = round(sum(Quantity) / 1024, 1) by DataType
| order by GB desc
| take 15
```

Find noisy sources within a table:

```kusto
ContainerLogV2
| where TimeGenerated > ago(1d)
| summarize GB = sum(_BilledSize) / pow(1024, 3) by PodNamespace, ContainerName
| top 10 by GB
```

Workspace transformation (DCR) dropping health-probe requests from App Service HTTP logs and removing a column:

```kusto
source
| where CsUriStem !in ("/healthz", "/ready")
| project-away CsCookie
```

Set a commitment tier and a daily cap (non-production workspace):

```bash
az monitor log-analytics workspace update -g rg-mon -n law-platform-prod --sku CapacityReservation --capacity-reservation-level 200
az monitor log-analytics workspace update -g rg-mon -n law-dev --quota 5        # 5 GB/day cap
```

Application Insights cost: tune sampling, avoid logging every request body, use log levels, and set a daily cap only as a safety net.

Govern it: monthly ingestion review per table and per team (tag workspaces and use `_ResourceId` for chargeback), anomaly alerts on daily ingestion, and budgets in Cost Management (Topic 29).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Value first** | Classify each source by detect/troubleshoot/comply before choosing plans. |
| **Never drop what you alert on** | Keep alerting signals in Analytics or metrics. |
| **Daily caps** | Useful on dev/test; risky on production (data loss during incidents). |
| **Commitment tiers** | When stable Analytics ingestion exceeds ~100 GB/day; review quarterly. |
| **Centralisation** | Consolidated workspaces reach commitment tiers sooner. |
| **Chargeback** | Show teams their ingestion to change behaviour. |

## Hands-on

1. Run the `Usage` query; identify the top three tables.
2. Add a DCR transformation that drops health probes and a sensitive column; compare daily volume.
3. Move a verbose table to Basic and another to Auxiliary; check alert impact.
4. Create a summary rule aggregating hourly error counts from a Basic table into an Analytics table, then alert on it.
5. Model commitment tier savings for your Analytics ingestion.

## Security, cost and reliability notes

- **Don't drop security-relevant logs** to save money without security sign-off.
- **Daily cap data loss** can hide incidents; monitor cap events.
- **Masking** with transformations also reduces privacy risk.
- **Sentinel** adds its own pricing on top of Log Analytics; plan both together.

## Common mistakes

| Mistake | Fix |
|---|---|
| Everything in Analytics plan | Plan by value |
| Debug logging in production | Proper log levels |
| 100% trace sampling on high-volume apps | Sampling |
| Commitment tier set and forgotten | Quarterly review |
| No ingestion monitoring | Usage queries and anomaly alerts |

## Check yourself

1. What are the main cost drivers in Azure Monitor?
2. Name six cost levers.
3. What do commitment tiers cover and save?
4. When is a daily cap appropriate?
5. How do DCR transformations reduce cost?
6. How do you find the most expensive data?

<details>
<summary>Answers</summary>

1. Log ingestion, retention, queries on Basic/Auxiliary, metrics ingestion, alerts and feature add-ons such as Sentinel.
2. Any six: collect less at source, DCR transformations, sampling, table plans, retention, summary rules, commitment tiers, dedicated clusters, daily cap, Sentinel data lake.
3. Analytics Logs ingestion only; up to about 30% savings versus pay-as-you-go, from 100 GB/day, with a 31-day commitment.
4. On non-critical workspaces as a safety net; risky in production because data beyond the cap is lost.
5. By filtering rows and removing columns at ingestion so they're never billed.
6. Query the `Usage` table by `DataType` and use `_BilledSize` within tables.

</details>

## References

- [Azure Monitor Logs cost calculations](https://learn.microsoft.com/azure/azure-monitor/logs/cost-logs)
- [Cost optimisation in Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/fundamentals/best-practices-cost)
- [Table plans](https://learn.microsoft.com/azure/azure-monitor/logs/logs-table-plans)
- [Summary rules](https://learn.microsoft.com/azure/azure-monitor/logs/summary-rules)
- [Sampling in Application Insights](https://learn.microsoft.com/azure/azure-monitor/app/sampling)
