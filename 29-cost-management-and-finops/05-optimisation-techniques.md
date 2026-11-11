# Optimisation Techniques

## Why it matters

Industry surveys consistently find a large share of cloud spend is wasted: idle resources, oversized VMs, dev environments running all weekend, premium storage holding cold data, logs nobody queries. **Usage optimisation** (using less, or using cheaper resources for the same outcome) complements **rate optimisation** (commitment discounts, file 03) and should come first, so you don't commit to waste. The Azure Well-Architected cost optimisation pillar, Azure Advisor and Cost Management provide the guidance and recommendations; the work is turning them into habits and automation.

## Key concepts

### Optimisation levers

| Lever | Examples |
|---|---|
| **Eliminate waste** | Unattached disks, orphaned public IPs and NICs, empty App Service plans, idle load balancers, old snapshots, unused PTU deployments, forgotten test environments |
| **Rightsize** | Downsize VMs, databases, App Service plans, AKS node pools based on utilisation; Advisor rightsizing recommendations |
| **Schedule** | Stop/deallocate dev/test outside working hours (start/stop automation, Dev/Test auto-shutdown), scale non-production to zero |
| **Autoscale** | Scale out/in with demand (VMSS, App Service, AKS cluster autoscaler/KEDA, Container Apps scale-to-zero) |
| **Choose the right service model** | Serverless/consumption for spiky workloads; PaaS over IaaS to cut operations; containers to improve density |
| **Spot** | Batch, CI runners, stateless scale-out on spot VMs/node pools with eviction handling |
| **Storage tiering** | Hot → cool → cold → archive with lifecycle management; right redundancy (LRS for re-creatable data) |
| **Data transfer** | Keep traffic in-region, use private connectivity wisely, cache with Front Door/CDN, compress |
| **Observability tuning** | Log plans, sampling, retention (Topic 26 file 10) |
| **Licensing** | Azure Hybrid Benefit, dev/test pricing for non-production subscriptions |
| **Architecture** | Caching, queue-based load levelling, efficient queries, right-sized AI models (file 06) |

### Recommendation sources

| Source | Provides |
|---|---|
| **Azure Advisor (cost)** | Rightsize/shut down underused VMs (configurable CPU threshold), reservation and savings plan purchases, idle resources, App Service and SQL recommendations |
| **Cost Management** | Spending trends, anomalies, optimisation workbook |
| **Azure Optimization Engine / FinOps toolkit** | Extended recommendations and workbooks (file 07) |
| **Service-specific insights** | AKS cost analysis, Storage insights, SQL Database Advisor |
| **Well-Architected cost optimisation** | Design principles and checklist |

### Rightsizing method

1. Collect utilisation over representative periods (at least 2–4 weeks, including month-end peaks).
2. Look at CPU, memory, disk and network together (memory needs guest metrics).
3. Pick the smallest size meeting P95 needs with headroom; consider newer generations (better price/performance) and ARM-based SKUs.
4. Change in non-production first; monitor after change.

## How it works on Azure

Start/stop for non-production VMs with Azure Functions or Automation using tags (`autoShutdown=19:00`, `autoStart=07:00`), or Azure's start/stop VMs solution; DevTest Labs and Dev/Test policies for auto-shutdown.

Storage lifecycle policy:

```json
{
  "rules": [ {
    "name": "tier-logs",
    "enabled": true,
    "type": "Lifecycle",
    "definition": {
      "filters": { "blobTypes": [ "blockBlob" ], "prefixMatch": [ "logs/" ] },
      "actions": { "baseBlob": {
        "tierToCool": { "daysAfterModificationGreaterThan": 30 },
        "tierToCold": { "daysAfterModificationGreaterThan": 90 },
        "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
        "delete": { "daysAfterModificationGreaterThan": 2555 } } }
    }
  } ]
}
```

Waste-finding queries (Resource Graph):

```kusto
// Orphaned public IPs
resources
| where type =~ "microsoft.network/publicipaddresses" and isnull(properties.ipConfiguration) and isnull(properties.natGateway)
| project name, resourceGroup, sku = tostring(sku.name)

// Empty App Service plans
resources
| where type =~ "microsoft.web/serverfarms" and toint(properties.numberOfSites) == 0
| project name, resourceGroup, sku = tostring(sku.name)
```

Advisor cost recommendations in bulk:

```kusto
advisorresources
| where type == "microsoft.advisor/recommendations" and properties.category == "Cost"
| extend savings = todouble(properties.extendedProperties.savingsAmount), problem = tostring(properties.shortDescription.problem)
| summarize totalSavings = sum(savings), count() by problem
| order by totalSavings desc
```

Spot node pool on AKS for batch workloads:

```bash
az aks nodepool add -g rg-aks -n spotpool --cluster-name aks-prod --priority Spot --eviction-policy Delete \
  --spot-max-price -1 --enable-cluster-autoscaler --min-count 0 --max-count 20 --node-vm-size Standard_D8s_v5
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Cost vs reliability** | Don't remove redundancy needed for SLOs; optimise within the target (Topic 28). |
| **Cost vs performance** | Rightsize with headroom; measure user impact after changes. |
| **Automation** | Automate recurring savings (schedules, lifecycle, autoscale) rather than one-off clean-ups. |
| **Ownership** | Route recommendations to resource owners with due dates (governance rules, tickets). |
| **Prioritisation** | Biggest savings with lowest effort and risk first. |
| **Unit economics** | Track cost per transaction/customer to judge efficiency as you grow (file 07). |

## Hands-on

1. Run the waste queries and estimate savings; delete or justify each item.
2. Apply three Advisor rightsizing recommendations in non-production and monitor.
3. Implement auto-shutdown for dev VMs with tags.
4. Add a storage lifecycle policy to a logs container.
5. Add a spot node pool for a batch workload with eviction handling.

## Security, cost and reliability notes

- **Deleting resources** can destroy data or break dependencies; check locks, backups and owners first.
- **Spot eviction**: design for interruption; not for stateful critical services.
- **Archive tier** has rehydration time and early deletion fees.
- **Autoscale** needs sensible limits to avoid runaway scale-out costs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rightsizing on CPU only | Include memory, disk, network |
| One-off clean-up campaigns | Continuous automation and ownership |
| Committing before optimising | Optimise usage first |
| Cutting redundancy to save money | Respect SLOs |
| Ignoring non-compute costs | Storage, network, logs, AI tokens |

## Check yourself

1. List eight usage optimisation levers.
2. Describe a sound rightsizing method.
3. Which sources provide cost recommendations?
4. How do storage lifecycle policies save money?
5. When are spot VMs appropriate, and what must you handle?
6. Why optimise usage before buying commitments?

<details>
<summary>Answers</summary>

1. Any eight: eliminate waste, rightsize, schedule, autoscale, choose service model, spot, storage tiering, data transfer, observability tuning, licensing, architecture efficiency.
2. Collect 2–4 weeks of CPU, memory, disk and network data including peaks; choose the smallest size meeting P95 with headroom; test in non-production; monitor.
3. Azure Advisor, Cost Management, Azure Optimization Engine/FinOps toolkit, service-specific insights, Well-Architected guidance.
4. They move data to cheaper tiers as it ages and delete it after retention.
5. For interruption-tolerant workloads; handle evictions (checkpointing, retries, scale-out elsewhere).
6. To avoid committing to waste and over-buying.

</details>

## References

- [Well-Architected cost optimisation](https://learn.microsoft.com/azure/well-architected/cost-optimization/)
- [Advisor cost recommendations](https://learn.microsoft.com/azure/advisor/advisor-reference-cost-recommendations)
- [Spot virtual machines](https://learn.microsoft.com/azure/virtual-machines/spot-vms)
- [Blob lifecycle management](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
- [AKS cost analysis](https://learn.microsoft.com/azure/aks/cost-analysis)
