# Log Analytics and KQL

## Why it matters

Almost every investigation on Azure ends in a Log Analytics query: why did requests fail, who changed the firewall rule, which pods restarted, how many tokens did the agent use yesterday. **Log Analytics workspaces** store logs from resources, agents, Application Insights, Sentinel and custom sources, and **Kusto Query Language (KQL)** is how you ask questions of them. Workspace design decides who can see what, where data resides and how much you pay; table plans and retention decide the trade-off between query capability and cost. Fluency in essential KQL is one of the highest-leverage skills for engineers and architects on Azure.

## Key concepts

### Workspace design

| Decision | Options |
|---|---|
| **Topology** | Centralised (one workspace per tenant or region), decentralised (per team/app), or hybrid (central platform/security + app workspaces) |
| **Drivers for separation** | Data residency (region), sovereignty, billing separation, strict access isolation, different retention policies, Sentinel scope |
| **Access** | Workspace-context RBAC (Log Analytics Reader/Contributor), **resource-context** access (users see logs for resources they can access), **table-level RBAC** |
| **Cross-workspace queries** | `workspace("name").Table` and `union`; limit for performance |

Microsoft's guidance favours as few workspaces as possible: easier correlation, commitment tier savings and simpler management.

### Table plans

| Plan | Purpose | Query | Alerts | Cost |
|---|---|---|---|---|
| **Analytics** | Continuous monitoring, latency-sensitive analysis | Full KQL, interactive | Yes | Highest ingestion price |
| **Basic** | Troubleshooting and incident response | KQL with limits; query billed per GB scanned; 30-day interactive window | Limited | Lower ingestion |
| **Auxiliary / Lake** | High-volume, verbose data for long-term, inexpensive storage | Queryable for full retention, slower; billed per query | No | Lowest ingestion |

Azure tables default to Analytics; DCR-based custom tables support all plans; Basic/Auxiliary support varies by Azure table. Plan changes are limited to one per table per week. Moving a table to Auxiliary stops alerts on it.

### Retention

| Setting | Detail |
|---|---|
| **Analytics (interactive) retention** | 31 days included in ingestion price (90 days for Sentinel/Application Insights tables in some cases); extend up to 2 years |
| **Total retention** | Up to **12 years**; beyond interactive retention, data is in low-cost **long-term retention** |
| **Accessing long-term data** | **Search jobs** (results into a new table) or **restore** for intensive analysis |
| **Shortening retention** | Data removal waits 30 days so you can revert |

### KQL essentials

| Operator | Purpose |
|---|---|
| `where` | Filter rows |
| `project` / `extend` | Select / compute columns |
| `summarize ... by` | Aggregate (`count()`, `avg()`, `percentile()`, `dcount()`, `make_set()`) |
| `bin()` | Time buckets |
| `render` | Charts (timechart, barchart) |
| `join` / `lookup` | Combine tables |
| `parse` / `extract` / `parse_json` | Parse text and JSON |
| `top` / `order by` | Rank |
| `let` | Variables and reusable subqueries |
| `mv-expand` | Expand arrays |
| `ago()`, `now()`, `between` | Time filters |

Always filter by time first and project only needed columns for performance and Basic/Auxiliary cost.

## How it works on Azure

Common queries:

```kusto
// Top failing operations in App Service HTTP logs (resource-specific table)
AppServiceHTTPLogs
| where TimeGenerated > ago(1h) and ScStatus >= 500
| summarize failures = count() by CsUriStem, ScStatus
| top 10 by failures

// p95 latency trend for Application Insights requests
AppRequests
| where TimeGenerated > ago(24h)
| summarize p95 = percentile(DurationMs, 95), requests = count() by bin(TimeGenerated, 15m), Name
| render timechart

// Who deleted resources?
AzureActivity
| where TimeGenerated > ago(7d) and OperationNameValue endswith "/DELETE" and ActivityStatusValue == "Success"
| project TimeGenerated, Caller, ResourceGroup, Resource = tostring(split(_ResourceId, "/")[-1])

// Heartbeat gaps: machines silent for 15 minutes
Heartbeat
| summarize lastSeen = max(TimeGenerated) by Computer
| where lastSeen < ago(15m)

// Ingestion volume by table (cost analysis)
Usage
| where TimeGenerated > ago(30d) and IsBillable
| summarize GB = sum(Quantity) / 1024 by DataType
| order by GB desc
```

Configure table plan and retention with the CLI:

```bash
az monitor log-analytics workspace table update -g rg-mon -n ContainerLogV2 \
  --workspace-name law-platform-prod --plan Basic --total-retention-time 180

az monitor log-analytics workspace table update -g rg-mon -n AzureActivity \
  --workspace-name law-platform-prod --retention-time 90 --total-retention-time 730
```

**Summary rules** aggregate high-volume data (for example Basic/Auxiliary) into small Analytics tables on a schedule, enabling cheap dashboards and alerts on summarised data.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Central vs per-app workspaces** | Central for platform and security; separate only for clear drivers. |
| **Plan per table** | Analytics for alerting and frequent queries; Basic for debug logs; Auxiliary for verbose, compliance-driven data. |
| **Retention** | Interactive for operational window; total retention for compliance; avoid paying interactive prices for archive data. |
| **Access model** | Resource-context for app teams; table RBAC for sensitive tables. |
| **Query hygiene** | Saved queries and functions; avoid `search *` in production dashboards. |

## Hands-on

1. Create a workspace; connect App Service and Application Insights.
2. Write the five queries above against your data and save them as functions.
3. Move a verbose custom table to Basic; run a query and note the billed data scanned.
4. Set total retention to 2 years for `AzureActivity`; run a search job on older data.
5. Configure resource-context RBAC so an app team sees only their resources' logs.

## Security, cost and reliability notes

- **Sensitive data in logs**: mask with DCR transformations; restrict with table RBAC.
- **Cost**: ingestion dominates; Basic/Auxiliary queries are billed per GB scanned.
- **Query limits**: large queries can time out; filter early, use summary rules.
- **Workspace region**: data residency and cross-region ingestion costs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Many tiny workspaces | Consolidate |
| Alerting on Auxiliary tables | Keep alerting data in Analytics |
| Queries without time filters | Filter by time first |
| Paying interactive retention for compliance data | Long-term retention |
| Everyone with workspace-wide read access | Resource-context and table RBAC |

## Check yourself

1. When should you create additional workspaces?
2. Compare Analytics, Basic and Auxiliary table plans.
3. How long can data be retained, and how do you query old data?
4. Name eight essential KQL operators.
5. What are summary rules for?
6. How can app teams see only their own logs?

<details>
<summary>Answers</summary>

1. For data residency, sovereignty, billing separation, strict access isolation or different retention needs.
2. Analytics: full KQL and alerts, highest cost. Basic: lower cost, limited KQL, per-GB query billing, 30-day query window. Auxiliary/Lake: cheapest, for verbose long-term data, queries billed, no alerts.
3. Up to 12 years total; use search jobs or restore for long-term data.
4. where, project, extend, summarize, bin, join, parse/parse_json, render, top, let, mv-expand.
5. Aggregating high-volume data into small Analytics tables for cheap dashboards and alerts.
6. Resource-context access, plus table-level RBAC.

</details>

## References

- [Table plans](https://learn.microsoft.com/azure/azure-monitor/logs/logs-table-plans)
- [Data retention](https://learn.microsoft.com/azure/azure-monitor/logs/data-retention-configure)
- [Query Basic and Auxiliary tables](https://learn.microsoft.com/azure/azure-monitor/logs/basic-logs-query)
- [Design a Log Analytics workspace architecture](https://learn.microsoft.com/azure/azure-monitor/logs/workspace-design)
- [KQL overview](https://learn.microsoft.com/kusto/query/)
