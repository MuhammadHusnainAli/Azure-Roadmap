# Availability Zones and Multi-Region Design

## Why it matters

Datacentres lose power, cooling and network; whole regions occasionally have major incidents. Azure gives you two levels of fault isolation: **availability zones** within a region and **multiple regions**. Most workloads get excellent resilience from zone redundancy alone, at modest cost and complexity. Multi-region designs protect against regional outages and support global users, but they introduce data consistency, failover and cost challenges that must be justified by business requirements. Choosing correctly, service by service, is one of the most consequential architecture decisions on Azure.

## Key concepts

### Availability zones

An availability zone is a logical grouping of one or more physically separate datacentres in a region, typically several kilometres apart and usually within 100 km, with independent power, cooling and networking and low-latency links. Most Azure regions have three zones.

| Deployment | Meaning | Who handles zone failure |
|---|---|---|
| **Zone-redundant** | Service replicates/distributes across zones (for example ZRS storage, zone-redundant SQL, App Service with zone redundancy, Front Door, Container Apps environments) | **Microsoft** fails over automatically |
| **Zonal** | Resource pinned to one zone you choose (VM, managed disk, zonal IP) | **You** must deploy instances in multiple zones and route between them |
| **Non-zonal / regional** | No zone guarantee | Could be affected by any zone outage |

Zone-redundant or multi-zone zonal deployments are **zone-resilient**. Logical zone numbers map to physical zones differently per subscription; use the `availabilityZoneMappings` API when coordinating across subscriptions.

### Multi-region

| Pattern | Description | RTO/RPO | Cost |
|---|---|---|---|
| **Backup and restore** | Restore to another region from geo-redundant backups | Hours/hours | Lowest |
| **Pilot light / warm standby** | Data replicated; minimal or scaled-down compute in secondary; scale up on failover | Minutes–hours / seconds–minutes | Medium |
| **Active-passive (hot standby)** | Full stack in secondary, receiving no user traffic | Minutes / seconds | High |
| **Active-active** | Both regions serve traffic; global load balancing | Near zero / near zero (with multi-write or conflict handling) | Highest; most complex |

**Deployment stamps**: identical, independently deployable units (per region or per tenant group) that scale out and contain failures.

### Region selection

| Factor | Consider |
|---|---|
| Data residency and sovereignty | Regulations, customer contracts |
| Service and SKU availability | Especially AI models, GPUs, PTU capacity |
| Availability zones | Prefer regions with zones |
| Latency to users and dependencies | Global routing |
| Paired regions | Some services' geo-replication and platform recovery use pairs (GRS storage, sequential updates); many newer regions are **nonpaired** and services offer choose-your-own secondary |
| Cost | Prices vary by region |
| Capacity | Large or specialised SKUs may be constrained |

### Global routing

| Service | Use |
|---|---|
| **Azure Front Door** | Global HTTP(S) load balancing with health probes, WAF, caching; active-active or priority failover |
| **Traffic Manager** | DNS-based routing (priority, weighted, performance, geographic) for any protocol |
| **Cross-region Load Balancer** | Global layer-4 for regional public load balancers |
| **API Management (multi-region gateways)** | Global API front door (Premium) |

### Data is the hard part

Stateless compute is easy to duplicate; data needs replication with a consistency model: synchronous (zone-redundant, RPO 0, latency cost) vs asynchronous (geo-replication, RPO > 0). Multi-write (Cosmos DB) needs conflict resolution. Design the data tier first (file 05).

## How it works on Azure

Zone-redundant baseline (Bicep fragments):

```bicep
resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-orders-prod'
  location: location
  sku: { name: 'P1v3', capacity: 3 }
  properties: { zoneRedundant: true }
}

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'storderprod'
  location: location
  sku: { name: 'Standard_GZRS' }          // zone-redundant + geo-redundant to secondary
  kind: 'StorageV2'
}

resource sql 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  name: '${sqlServer.name}/orders'
  location: location
  sku: { name: 'GP_Gen5', tier: 'GeneralPurpose', capacity: 4 }
  properties: { zoneRedundant: true }
}
```

Active-passive multi-region for a web workload:

```text
Front Door (priority routing, health probes) ─► Region A (primary): zone-redundant App Service, SQL (failover group primary), Redis, Storage GZRS
                                           └─► Region B (secondary): App Service scaled down, SQL failover group secondary, Storage secondary
Failover: SQL failover group switch (automatic or manual) → Front Door routes to B as A's health probes fail → scale out B
```

Check zone support and mappings:

```bash
az vm list-skus -l westeurope --zone --size Standard_D4s_v5 -o table
az rest --method get --url "https://management.azure.com/subscriptions/<sub>/locations?api-version=2022-12-01" \
  --query "value[?name=='westeurope'].availabilityZoneMappings"
```

Use Azure's "multiregion support" reference to see which services provide built-in geo-replication between paired or nonpaired regions, and which require your own design.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Zones first** | Make every production workload zone-resilient; it's usually the best reliability-per-cost step. |
| **Multi-region only when needed** | Regional outage tolerance, regulatory requirements or global latency; otherwise backup-and-restore DR. |
| **Active-active vs active-passive** | Active-active for near-zero RTO and global users if data supports it; active-passive is simpler. |
| **Failover automation** | Automate detection and routing; keep a human decision for data-tier failovers with data loss risk. |
| **Symmetry** | Secondary region must have quota and capacity (reserve capacity or capacity reservations for critical VMs/PTUs). |
| **Dependencies** | Identity, DNS, Key Vault, container registries, AI models and secrets must be available in both regions. |

## Hands-on

1. Audit a workload: list each resource's zone support (zone-redundant, zonal, none).
2. Redeploy App Service, Storage and SQL as zone-redundant with Bicep.
3. Build an active-passive setup with Front Door priority routing and a SQL failover group.
4. Simulate primary failure (stop the primary app) and measure failover time.
5. Document region selection for a regulated workload, including AI model availability.

## Security, cost and reliability notes

- **Zone redundancy cost** is often small (some services charge more or need minimum instances).
- **Multi-region doubles** much of the bill; warm standby reduces it.
- **Security controls** (policies, private endpoints, Key Vault, Defender) must be deployed in both regions.
- **Test failover regularly**; untested failover often fails.

## Common mistakes

| Mistake | Fix |
|---|---|
| Single-instance zonal VMs believed to be "zone-resilient" | Multiple zones with load balancing, or zone-redundant services |
| Multi-region compute with single-region data | Replicate data tier |
| Assuming paired-region failover will happen automatically for everything | Know each service's behaviour |
| No capacity in the secondary region | Quota and capacity reservations |
| Forgetting global dependencies | Map identity, DNS, registries, keys |

## Check yourself

1. What is an availability zone?
2. Compare zone-redundant and zonal deployments.
3. Describe four multi-region patterns with RTO/RPO trade-offs.
4. What factors drive region selection?
5. Which services provide global traffic routing?
6. Why is data the hardest part of multi-region design?

<details>
<summary>Answers</summary>

1. A logical group of physically separate datacentres within a region with independent power, cooling and networking.
2. Zone-redundant: the service spreads across zones and Microsoft handles failover. Zonal: pinned to one zone; you deploy across zones and handle failover.
3. Backup/restore (hours, lowest cost), warm standby (minutes–hours, medium), active-passive hot standby (minutes, high), active-active (near zero, highest).
4. Residency, service/SKU availability, zones, latency, pairing, cost, capacity.
5. Front Door, Traffic Manager, cross-region Load Balancer, multi-region API Management.
6. Replication involves consistency, latency and data-loss trade-offs, and multi-write needs conflict handling.

</details>

## References

- [What are availability zones?](https://learn.microsoft.com/azure/reliability/availability-zones-overview)
- [Azure services that support multiple regions](https://learn.microsoft.com/azure/reliability/regions-multiregion-support)
- [Azure region pairs and nonpaired regions](https://learn.microsoft.com/azure/reliability/cross-region-replication-azure)
- [Multi-region design (Well-Architected)](https://learn.microsoft.com/azure/well-architected/reliability/highly-available-multi-region-design)
- [Deployment stamps pattern](https://learn.microsoft.com/azure/architecture/patterns/deployment-stamp)
