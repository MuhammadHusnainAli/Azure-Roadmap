# 26 · Monitoring and Observability

> Tracks: E ● A ● C ○ · Part E · Secure, Operate, Migrate and Extend

Operating Azure workloads well depends on seeing them clearly. This topic covers Azure Monitor's data platform (metrics, logs, traces, data collection rules); Log Analytics workspace design, table plans, retention and essential KQL; Application Insights with the Microsoft OpenTelemetry Distro, distributed tracing and standard availability tests; alerts, dynamic thresholds, action groups and alert processing rules; workbooks, Azure Monitor dashboards with Grafana and Azure Managed Grafana; Kubernetes monitoring with managed Prometheus and container logs; VM and network monitoring with the Azure Monitor Agent, VNet flow logs and Network Watcher; Service Health, Resource Health and Advisor; operational monitoring for AI workloads; and controlling observability cost.

## What you will be able to do

- Explain Azure Monitor's data types and how each is collected.
- Design Log Analytics workspaces, table plans and retention; write essential KQL.
- Instrument applications with OpenTelemetry and trace requests end to end.
- Build actionable, owned alerts with dynamic thresholds and processing rules.
- Create workbooks and Grafana dashboards as code for different audiences.
- Monitor AKS with managed Prometheus, container logs and control plane logs.
- Monitor VMs and networks with AMA, DCRs, VNet flow logs and Connection Monitor.
- Use Service Health, Resource Health and Advisor in operations and governance.
- Monitor AI workloads for tokens, latency, throttling, PTU and agent behaviour.
- Control observability cost without losing investigative capability.

## Prerequisites

[Topic 03](../03-governance-and-landing-zones/README.md) (policy), [Topic 05](../05-networking/README.md), [Topic 06](../06-compute/README.md) and [Topic 07](../07-containers-and-kubernetes/README.md) for the resources being monitored. [Topic 22](../22-evaluation-observability-and-genaiops/README.md) covers AI quality monitoring.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure Monitor Overview](01-azure-monitor-overview.md) | 35 min | Diagnostic settings via policy |
| 02 | [Log Analytics and KQL](02-log-analytics-and-kql.md) | 50 min | Queries, table plans, retention |
| 03 | [Application Insights](03-application-insights.md) | 45 min | OpenTelemetry distro, availability test |
| 04 | [Alerts and Action Groups](04-alerts-and-action-groups.md) | 40 min | Dynamic threshold alert, processing rule |
| 05 | [Workbooks and Dashboards](05-workbooks-and-dashboards.md) | 40 min | SLO workbook, Grafana |
| 06 | [Container and Kubernetes Monitoring](06-container-and-kubernetes-monitoring.md) | 45 min | Managed Prometheus on AKS |
| 07 | [VM and Network Monitoring](07-vm-and-network-monitoring.md) | 45 min | DCRs, VNet flow logs |
| 08 | [Service Health, Resource Health and Advisor](08-service-health-resource-health-and-advisor.md) | 30 min | Service Health alerts, Advisor query |
| 09 | [Monitoring AI Workloads](09-monitoring-ai-workloads.md) | 40 min | AI alerts and workbook |
| 10 | [Observability Cost Control](10-observability-cost-control.md) | 35 min | Usage analysis, transformations |

**AI engineers:** 01–04 and 09. **Architects:** all of it, especially 01, 02, 04 and 10. **Consultants:** 01, 08 and 10.

## Completion checklist

- [ ] I can explain metrics vs logs vs traces and how DCRs and diagnostic settings collect data.
- [ ] I have designed a workspace topology and configured table plans and retention.
- [ ] I have instrumented an app with OpenTelemetry and viewed its application map.
- [ ] I have built alerts with dynamic thresholds, action groups and maintenance suppression.
- [ ] I have built a workbook and a Grafana dashboard and stored them as code.
- [ ] I have enabled managed Prometheus and container logs on AKS.
- [ ] I have deployed AMA with DCRs and enabled VNet flow logs.
- [ ] I have Service Health and Resource Health alerts and an Advisor review.
- [ ] I have alerts for 429s, TTFT and PTU utilisation on an AI deployment.
- [ ] I have reduced ingestion cost with transformations, plans or sampling.

## Maps to

- **AZ-104**: monitor and maintain Azure resources.
- **AZ-305**: design monitoring and logging solutions.
- **AZ-400** (adjacent): instrumentation and feedback.

## Next

27 · [DevOps and Platform Engineering](../27-devops-and-platform-engineering/README.md)
