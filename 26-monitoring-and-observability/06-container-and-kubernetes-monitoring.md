# Container and Kubernetes Monitoring

## Why it matters

Kubernetes adds layers that traditional monitoring misses: nodes, pods, containers, the control plane, cluster networking and workloads that move constantly. A failing pod gets rescheduled before anyone notices; a noisy neighbour exhausts node memory; a misconfigured HPA thrashes; the API server throttles. The cloud-native standard for Kubernetes metrics is **Prometheus**, with **Grafana** for dashboards. Azure provides both as managed services (**Azure Monitor managed service for Prometheus** and **Azure Managed Grafana** or portal Grafana dashboards) plus **container log collection** (Container insights) and control-plane logs, so AKS and Arc-enabled clusters get production-grade observability without running monitoring infrastructure yourself.

## Key concepts

### Monitoring layers and services

| Layer | What to watch | Azure service |
|---|---|---|
| **Network** | VNet flows, load balancer health, DNS | Network Watcher, traffic analytics, Network insights |
| **Cluster infrastructure and nodes** | Node CPU/memory/disk, kubelet, node pools | Managed Prometheus (node exporter, kube-state-metrics), Container insights |
| **Control plane** | API server, scheduler, controller manager, etcd, audit logs | Control plane **resource logs** via diagnostic settings; managed Prometheus control plane metrics |
| **Workloads** | Pod restarts, OOMKills, pending pods, HPA, deployments | Prometheus metrics, Kubernetes events |
| **Container logs** | stdout/stderr | Container log collection (containerised Azure Monitor Agent) into `ContainerLogV2` |
| **Applications** | Requests, dependencies, traces | Application Insights / OpenTelemetry (file 03) |

### Azure Monitor managed service for Prometheus

| Feature | Detail |
|---|---|
| **Storage** | **Azure Monitor workspace** (separate from Log Analytics workspace) |
| **Query** | Full PromQL in the portal and Grafana |
| **Retention** | 18 months, no storage cost; pricing based on ingestion and queries |
| **Content** | Preconfigured scrape configs, recording rules, alert rules and Grafana dashboards |
| **Sources** | AKS, Arc-enabled Kubernetes, VMs; **remote write** from self-managed Prometheus |
| **Alerting** | Prometheus alert rule groups (PromQL) firing into Azure Monitor action groups |

### Container log collection (Container insights)

Collects stdout/stderr and Kubernetes events through a containerised Azure Monitor Agent with DCRs. Use the **ContainerLogV2** schema, filter namespaces, and choose Analytics or Basic table plans for cost (file 10). Container insights views show nodes, controllers, containers and live logs.

### Visualisation

Grafana dashboards (portal or Managed Grafana) for Prometheus metrics (cluster, node, namespace, workload, kubelet, API server); workbooks for logs-based views.

## How it works on Azure

Enable managed Prometheus, container logs and Grafana integration on an AKS cluster:

```bash
az monitor account create -g rg-mon -n amw-contoso-prod -l westeurope          # Azure Monitor workspace
az aks update -g rg-aks -n aks-prod \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id $(az monitor account show -g rg-mon -n amw-contoso-prod --query id -o tsv) \
  --grafana-resource-id $(az grafana show -g rg-mon -n amg-contoso --query id -o tsv)
az aks enable-addons -g rg-aks -n aks-prod -a monitoring \
  --workspace-resource-id $(az monitor log-analytics workspace show -g rg-mon -n law-platform-prod --query id -o tsv)
```

Also create a diagnostic setting for control plane logs (`kube-audit-admin`, `kube-apiserver`, `cluster-autoscaler`) to the same Log Analytics workspace; avoid the very verbose `kube-audit` unless required.

PromQL examples:

```promql
# Pods restarting in the last 15 minutes
increase(kube_pod_container_status_restarts_total[15m]) > 0

# Node memory utilisation
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Pending pods per namespace
sum by (namespace) (kube_pod_status_phase{phase="Pending"})
```

KQL on container logs (errors by namespace and container):

```kusto
ContainerLogV2
| where TimeGenerated > ago(1h) and LogLevel in ("error", "critical")
| summarize errors = count() by PodNamespace, ContainerName
| order by errors desc
```

Example Prometheus alert rule (Bicep `Microsoft.AlertsManagement/prometheusRuleGroups`) condition: `kube_pod_container_status_restarts_total` increase > 3 in 10 minutes → Sev 2 to the platform action group. The recommended alert rule set can be enabled from the portal.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Managed vs self-hosted Prometheus** | Managed for no-ops scale and retention; remote-write from existing Prometheus during migration. |
| **Metrics vs logs** | Prometheus for health and capacity; logs for debugging. Don't parse metrics out of logs. |
| **Log volume** | Filter namespaces (kube-system noise), use ContainerLogV2 with Basic plan for verbose apps, sample debug logs. |
| **Workspace topology** | One Azure Monitor workspace per environment/region; shared Log Analytics workspace per platform. |
| **Control plane logs** | Collect `kube-audit-admin` rather than full `kube-audit` for cost. |
| **Multicluster** | Arc-enable non-AKS clusters for one monitoring approach. |

## Hands-on

1. Enable managed Prometheus, container logs and Grafana on an AKS cluster.
2. Open the Kubernetes Grafana dashboards; find the busiest namespace.
3. Deploy a pod that exhausts memory; observe OOMKill in metrics and events.
4. Create a Prometheus alert rule for restarts with an action group.
5. Switch `ContainerLogV2` to Basic plan for a noisy namespace and compare cost.

## Security, cost and reliability notes

- **Sensitive data in container logs**: avoid logging secrets; use DCR transformations to mask.
- **Cost**: Prometheus ingestion (samples) and log ingestion; drop unneeded metrics with scrape configs and filter logs.
- **Monitoring reliability**: managed services remove the risk of your Prometheus pods failing during cluster incidents.
- **RBAC**: Grafana and workspace access via Entra; restrict who can query logs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Collecting all container logs at Analytics plan | Filter and use Basic where appropriate |
| No control plane logs | Diagnostic settings |
| Self-hosting Prometheus without retention planning | Managed Prometheus |
| Alerts only on node CPU | Workload signals (restarts, pending, OOM, HPA) |
| Monitoring AKS only, not hybrid clusters | Arc-enabled Kubernetes |

## Check yourself

1. Which Azure services monitor each Kubernetes layer?
2. Where are managed Prometheus metrics stored, and for how long?
3. How are container logs collected, and which schema should you use?
4. How do you collect control plane logs?
5. Give two PromQL alert examples.
6. How can you reduce Kubernetes monitoring cost?

<details>
<summary>Answers</summary>

1. Network: Network Watcher/insights; nodes and workloads: managed Prometheus and container insights; control plane: resource logs; apps: Application Insights.
2. In an Azure Monitor workspace, retained for 18 months with no storage cost (ingestion and queries billed).
3. Containerised Azure Monitor Agent with DCRs into ContainerLogV2.
4. Diagnostic settings on the AKS resource sending control-plane categories to Log Analytics.
5. Pod restarts increase over a window; pending pods per namespace; node memory utilisation above a threshold.
6. Filter namespaces, use Basic plan for verbose logs, drop unneeded metrics, avoid full `kube-audit`.

</details>

## References

- [Kubernetes monitoring in Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/containers/kubernetes-monitoring-overview)
- [Azure Monitor managed service for Prometheus](https://learn.microsoft.com/azure/azure-monitor/metrics/prometheus-metrics-overview)
- [Azure Managed Grafana](https://learn.microsoft.com/azure/managed-grafana/overview)
- [Monitor AKS](https://learn.microsoft.com/azure/aks/monitor-aks)
