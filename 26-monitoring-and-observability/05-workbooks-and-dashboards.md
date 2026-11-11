# Workbooks and Dashboards

## Why it matters

During an incident, nobody has time to write KQL from scratch. Operations teams need views that answer the standard questions instantly (is the service healthy, what changed, where are errors concentrated, are we within SLO and budget), while leadership needs simple trend views. Azure offers several visualisation tools: **Azure Workbooks** for interactive, parameterised reports; **Azure Monitor dashboards with Grafana** built into the portal; **Azure Managed Grafana** for a full Grafana service; and **Power BI** for business reporting. Choosing the right tool per audience, and treating dashboards as versioned code, makes observability usable rather than decorative.

## Key concepts

### Tools compared

| Tool | Strengths | Data sources | Best for |
|---|---|---|---|
| **Azure Workbooks** | Interactive parameters, KQL, Resource Graph, metrics, ARM queries, links and actions; templates in insights | Logs, metrics, Resource Graph, ARM, Azure Data Explorer, custom endpoints | Runbook-style troubleshooting views, operational reports, investigation guides |
| **Azure Monitor dashboards with Grafana** | Grafana dashboards inside the Azure portal; Azure-managed templates auto-provisioned for common resources and AKS; import from Grafana gallery or JSON | Azure Monitor, Azure Data Explorer, Prometheus (Azure Monitor workspace) | Teams wanting Grafana visuals without running Grafana |
| **Azure Managed Grafana** | Full Grafana service with plugins, alerting, many data sources, Entra SSO, private endpoints | Azure and non-Azure (Prometheus, Loki, SQL, third party) | Cross-platform observability teams, multicloud |
| **Azure portal dashboards** (classic) | Pin tiles from many blades | Metrics, logs, resource tiles | Simple personal or shared overviews |
| **Power BI** | Business reporting, data modelling, sharing | Log Analytics/ADX via connectors | Executive and business KPIs |

### Managed Grafana tiers

| Tier | Status |
|---|---|
| **Standard** (X1, X2 sizes) | Recommended; SLA; full features |
| **Essential** | Deprecated; no new workspaces; existing ones deprecated by **31 March 2027** – migrate to Standard |

### Workbook building blocks

| Element | Use |
|---|---|
| **Parameters** | Subscription, resource, time range, environment pickers that drive all queries |
| **Queries** | KQL, metrics, Resource Graph, ARM |
| **Visualisations** | Grids, charts, tiles, maps, graphs, honeycomb |
| **Groups and tabs** | Organise views (overview, failures, performance, dependencies) |
| **Conditional visibility and links** | Drill-down from summary to detail; deep links into Application Insights and resources |

### Dashboard design principles

| Principle | Meaning |
|---|---|
| **Audience first** | On-call, service owner, platform team and executives need different views |
| **Golden signals** | Latency, traffic, errors, saturation per service |
| **SLO-centred** | Show SLI vs SLO and error budget burn |
| **Top-down drill** | Summary → service → component → trace |
| **Annotate change** | Deployments and incidents on the timeline |
| **Few panels** | Each panel answers a question; remove the rest |

## How it works on Azure

Workbook query example: SLO view for request success rate with a time-range parameter:

```kusto
AppRequests
| where TimeGenerated {TimeRange}
| where AppRoleName in ({Services})
| summarize total = count(), failed = countif(Success == false) by AppRoleName
| extend availability = round(100.0 * (total - failed) / total, 3), slo = 99.9
| extend errorBudgetUsedPct = round(100.0 * (100 - availability) / (100 - slo), 1)
| project AppRoleName, total, availability, slo, errorBudgetUsedPct
```

Resource Graph tile: resources missing required tags:

```kusto
resources
| where isnull(tags.owner) or isnull(tags.environment)
| summarize missing = count() by type
| order by missing desc
```

Workbooks as code: export the workbook's gallery template JSON and deploy with Bicep (`Microsoft.Insights/workbooks`) in the same repository as the service; Grafana dashboards as JSON in Git, deployed through the Grafana API or Azure CLI (`az grafana dashboard create`).

Managed Grafana with Entra and managed identity (CLI):

```bash
az grafana create -g rg-mon -n amg-contoso --sku-tier Standard
# Grafana's managed identity needs Monitoring Reader on subscriptions it visualises
az role assignment create --assignee $(az grafana show -g rg-mon -n amg-contoso --query identity.principalId -o tsv) \
  --role "Monitoring Reader" --scope /subscriptions/<sub-id>
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Workbooks vs Grafana** | Workbooks for Azure-native investigation with Resource Graph and actions; Grafana for Prometheus-heavy, multi-source and cross-cloud views. |
| **Portal Grafana vs Managed Grafana** | Portal dashboards with Grafana for no-ops Grafana visuals on Azure data; Managed Grafana for plugins, broader sources, Grafana alerting and team workflows. |
| **Ownership** | Each service team owns its dashboards; platform team owns shared templates. |
| **Versioning** | Store JSON in Git; review changes. |
| **Access** | Viewers need read on underlying data; use resource-context access. |
| **Performance** | Summarise in queries; use summary rules for heavy aggregates. |

## Hands-on

1. Build a workbook with subscription, resource group and time range parameters showing golden signals for an App Service app.
2. Add the SLO query and a drill-down link to Application Insights failures.
3. Open Azure Monitor dashboards with Grafana in the portal; use an Azure-managed AKS template; import a gallery dashboard.
4. Create a Managed Grafana Standard workspace; connect it to an Azure Monitor workspace (Prometheus) and Log Analytics.
5. Export the workbook template and deploy it with Bicep.

## Security, cost and reliability notes

- **Least privilege**: Grafana's managed identity needs only Monitoring Reader/Log Analytics Reader where required.
- **Network**: Managed Grafana supports private endpoints and managed private endpoints to data sources.
- **Cost**: queries against Basic/Auxiliary tables are billed; Managed Grafana Standard is billed per instance and active users.
- **Essential tier deprecation**: plan migration before 31 March 2027.

## Common mistakes

| Mistake | Fix |
|---|---|
| Dozens of unowned dashboards | Owners and periodic clean-up |
| Dashboards querying raw high-volume data | Summaries and summary rules |
| Manual portal-only dashboards | Dashboards as code |
| Same view for executives and engineers | Audience-specific views |
| No deployment annotations | Annotate releases |

## Check yourself

1. Compare Workbooks, Azure Monitor dashboards with Grafana, Managed Grafana and Power BI.
2. What happens to the Managed Grafana Essential tier?
3. What are the golden signals?
4. How do you version and deploy workbooks?
5. What permissions does Managed Grafana need to read Azure Monitor data?
6. Give three dashboard design principles.

<details>
<summary>Answers</summary>

1. Workbooks: interactive Azure-native reports with KQL, metrics, Resource Graph and actions. Portal Grafana dashboards: Grafana visuals on Azure Monitor, ADX and Prometheus data without running Grafana. Managed Grafana: full Grafana service with plugins and many sources. Power BI: business reporting.
2. No new Essential workspaces; existing ones are deprecated by 31 March 2027; migrate to Standard.
3. Latency, traffic, errors and saturation.
4. Export the template JSON to Git and deploy `Microsoft.Insights/workbooks` with Bicep or other IaC.
5. Monitoring Reader (and Log Analytics Reader where needed) on relevant scopes for its managed identity.
6. Any three: audience first, golden signals, SLO-centred, top-down drill-down, annotate change, few purposeful panels.

</details>

## References

- [Azure Workbooks overview](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview)
- [Azure Monitor dashboards with Grafana](https://learn.microsoft.com/azure/azure-monitor/visualize/visualize-use-grafana-dashboards)
- [Azure Managed Grafana overview](https://learn.microsoft.com/azure/managed-grafana/overview)
- [Create a dashboard from logs](https://learn.microsoft.com/azure/azure-monitor/visualize/tutorial-logs-dashboards)
