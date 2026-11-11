# Azure Monitor Overview

## Why it matters

You can't operate what you can't see. When a customer reports that "the app is slow", the on-call engineer needs to know within minutes whether the problem is the front end, an API, a database, a dependency, the network or the platform itself. **Azure Monitor** is Azure's unified observability platform: it collects metrics, logs, traces and changes from every layer (Azure platform, guest operating systems, containers, applications and AI services), stores them in purpose-built data stores, and provides analysis, visualisation, alerting and automation on top. Every other file in this topic builds on its data model, so getting the fundamentals right (what data exists, where it lands and how it's collected) avoids blind spots and runaway costs later.

## Key concepts

### Data types

| Type | What it is | Store | Typical use |
|---|---|---|---|
| **Metrics** | Numeric time series sampled at intervals (CPU %, requests/sec, token counts) | Azure Monitor Metrics (platform metrics retained 93 days); **Azure Monitor workspace** for Prometheus metrics | Fast charts, near-real-time alerts, autoscale |
| **Logs** | Structured records with many properties (events, audit, diagnostics, app logs) | **Log Analytics workspace** | Investigation, correlation, complex queries with KQL |
| **Traces** | Distributed request flows made of spans (OpenTelemetry) | Application Insights (stored in Log Analytics) | End-to-end latency and dependency analysis |
| **Changes** | Configuration changes to resources | Change Analysis / Resource Graph change history | "What changed before it broke?" |
| **Activity log** | Control-plane operations on subscriptions | Platform (90 days), exportable to Log Analytics | Audit, who-did-what |

### Data sources by layer

| Layer | Data | How collected |
|---|---|---|
| **Azure tenant** | Entra sign-in and audit logs | Entra diagnostic settings |
| **Subscription** | Activity log, Service Health | Built in; diagnostic settings for export |
| **Azure resources** | Platform metrics (automatic), resource logs | Resource **diagnostic settings** |
| **Guest OS** (VMs, Arc servers) | Performance counters, event logs, syslog, text logs | **Azure Monitor Agent** + **data collection rules (DCRs)** |
| **Containers** | Prometheus metrics, container logs, Kubernetes events | Managed Prometheus, Container insights (DCRs) |
| **Applications** | Requests, dependencies, exceptions, traces, custom metrics | **Application Insights** with OpenTelemetry |
| **Custom sources** | Any JSON/CSV logs | **Logs ingestion API** with DCRs |

### Data collection rules

A **DCR** defines what to collect, how to transform it (KQL transformations at ingestion: filter, parse, mask, drop columns) and where to send it. DCRs are used by the Azure Monitor Agent, Container insights, the Logs ingestion API and workspace transformations. They are Azure resources, deployable as IaC, and the main lever for both coverage and cost (file 10).

### Insights and curated experiences

Prebuilt monitoring experiences combine data and visualisations for specific services: Application Insights, Container insights, VM insights, Network insights, Storage insights, Key Vault insights, SQL insights and others. They're the fastest way to useful visibility, built on the same workspaces you can query directly.

### Analysis and response

| Capability | Purpose | File |
|---|---|---|
| **Metrics explorer** | Chart and split metrics | 01 |
| **Log Analytics + KQL** | Query logs and traces | 02 |
| **Application Insights** | APM, distributed tracing | 03 |
| **Alerts and action groups** | Notify and automate | 04 |
| **Workbooks, dashboards, Grafana** | Visualise | 05 |
| **Autoscale** | Scale on metrics | ([Topic 06](../06-compute/README.md)) |
| **Export** | Event Hubs, Storage, partner tools | Diagnostic settings, data export |

## How it works on Azure

Diagnostic settings send resource logs and metrics to a workspace. Bicep example for a Key Vault:

```bicep
resource law 'Microsoft.OperationalInsights/workspaces@2023-09-01' existing = { name: 'law-platform-prod' }
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = { name: 'kv-contoso-app-prod' }

resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'to-law'
  scope: kv
  properties: {
    workspaceId: law.id
    logAnalyticsDestinationType: 'Dedicated'      // resource-specific tables
    logs: [ { categoryGroup: 'audit', enabled: true }, { categoryGroup: 'allLogs', enabled: false } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```

At scale, use Azure Policy built-in initiatives ("Enable allLogs/audit category group resource logging") with `DeployIfNotExists` at management group scope, so every new resource is covered automatically. Azure Landing Zones include a central platform workspace and these policies ([Topic 03](../03-governance-and-landing-zones/README.md)).

Typical observability architecture:

```text
Resources ──diagnostic settings──┐
VMs/Arc ──AMA + DCRs─────────────┼──► Log Analytics workspace(s) ──► KQL, workbooks, alerts, Sentinel
Apps ──OpenTelemetry─► App Insights ┘
AKS ──Managed Prometheus─► Azure Monitor workspace ──► Grafana dashboards, Prometheus alerts
Platform metrics (automatic) ──► Metrics explorer, metric alerts, autoscale
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Metrics vs logs** | Metrics for fast, cheap alerting on numeric signals; logs for detail and correlation. Don't alert on logs when a metric exists. |
| **Workspace topology** | Few workspaces: central platform/security workspace plus per-application or per-region where needed for data residency, access or billing (file 02). |
| **Resource-specific tables** | Prefer `Dedicated` destination type for structured, cheaper-to-query tables. |
| **Coverage by policy** | Enforce diagnostic settings with Azure Policy rather than per-team manual setup. |
| **OpenTelemetry** | Standardise application instrumentation on OpenTelemetry for portability (file 03). |
| **Cost awareness** | Decide collection by value; use DCR transformations and table plans (file 10). |

## Hands-on

1. Create a Log Analytics workspace and send Activity Log and a Key Vault's audit logs to it.
2. Open Metrics explorer for a storage account; split transactions by API name.
3. Assign the built-in "audit category group" diagnostic settings policy initiative to a test resource group and remediate.
4. Query `AzureActivity` for resource deletions in the last 7 days.
5. Open Application Insights, VM insights and Container insights for sample resources and note which data store each uses.

## Security, cost and reliability notes

- **Access**: use workspace and table-level RBAC; logs often contain sensitive data.
- **Cost**: log ingestion is the main cost driver; metrics are mostly free for platform metrics.
- **Coverage gaps**: platform metrics are automatic, but resource logs are not; without diagnostic settings you have no logs.
- **Retention**: platform metrics 93 days; logs per workspace/table configuration (file 02).

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming resource logs are collected by default | Diagnostic settings via policy |
| One workspace per resource | Consolidate |
| Log alerts where metric alerts would do | Use metrics first |
| No application telemetry | Instrument with OpenTelemetry |
| Collecting everything at the highest tier | Plan by value (file 10) |

## Check yourself

1. Name Azure Monitor's main data types and their stores.
2. How are guest OS data and resource logs collected?
3. What is a data collection rule?
4. When would you use metrics rather than logs?
5. How do you guarantee diagnostic settings across an estate?
6. What are insights?

<details>
<summary>Answers</summary>

1. Metrics (Azure Monitor Metrics, Azure Monitor workspace for Prometheus), logs (Log Analytics workspace), traces (Application Insights in Log Analytics), changes, activity log.
2. Guest OS via Azure Monitor Agent with DCRs; resource logs via diagnostic settings.
3. A resource defining what data to collect, ingestion-time transformations and destinations.
4. For numeric signals needing fast, cheap alerting, charts and autoscale.
5. Azure Policy `DeployIfNotExists` initiatives at management group scope.
6. Curated monitoring experiences for specific services built on Azure Monitor data.

</details>

## References

- [Azure Monitor overview](https://learn.microsoft.com/azure/azure-monitor/fundamentals/overview)
- [Azure Monitor Logs](https://learn.microsoft.com/azure/azure-monitor/logs/data-platform-logs)
- [Data collection rules](https://learn.microsoft.com/azure/azure-monitor/essentials/data-collection-rule-overview)
- [Diagnostic settings](https://learn.microsoft.com/azure/azure-monitor/essentials/diagnostic-settings)
