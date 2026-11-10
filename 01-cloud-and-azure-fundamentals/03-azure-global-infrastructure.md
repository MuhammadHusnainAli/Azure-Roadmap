# Azure Global Infrastructure

## Why it matters

"Which region?" is the first question on almost every deployment, and the answer constrains latency, compliance, cost, resilience and which services you can use at all. For AI it is sharper still: model availability and quota vary by region, so the region you want may not have the model you need.

## Key concepts

### The physical hierarchy

```text
Geography            e.g. Europe, United States, India
└── Region           e.g. West Europe, Sweden Central
    └── Availability zone      1, 2, 3
        └── Data centre(s)
```

**Data centre.** A physical building with its own power, cooling and networking. You never choose one directly.

**Availability zone (AZ).** One or more data centres within a region, physically separate, with independent power, cooling and networking, connected by a low-latency network (designed for under 2 ms round trip). A zone failure should not take out the others. Zone-enabled regions have at least three zones.

**Region.** A set of data centres deployed within a latency-defined perimeter. You deploy resources to a region. The public cloud lists about 60 regions.

**Geography.** A market, usually a country or a broad area, that contains one or more regions and defines a data residency and compliance boundary.

### Availability zones

Services support zones in one of three ways:

| Type | Behaviour | Examples |
|---|---|---|
| **Zonal** | You pin the resource to a specific zone. You get isolation, and you build redundancy by deploying copies to other zones | A VM in zone 1, a zonal managed disk |
| **Zone-redundant** | The platform spreads or replicates the resource across zones automatically | ZRS storage, zone-redundant Azure SQL Database, Standard Load Balancer |
| **Non-regional** (always available) | Global services with no region dependency | Microsoft Entra ID, Azure DNS, Front Door, Traffic Manager |

Two details that catch people out:

- **Zone numbers are logical and per subscription.** Zone 1 in your subscription may be a different physical zone from zone 1 in another subscription. This matters when coordinating across subscriptions; use the physical zone mapping API when it does.
- **A zone-enabled region does not mean every service there supports zones.** Check the reliability guide for each service.

### Region pairs and non-paired regions

Many older regions are **paired** with another region in the same geography, typically at least 300 miles away. Pairing gives:

- **Sequenced platform updates:** planned updates roll out to one region of a pair at a time.
- **Recovery priority:** in a broad outage, one region of each pair is prioritised.
- **Default replication target:** geo-redundant storage (GRS) replicates to the paired region, and you cannot choose another.

Things to know about pairs:

- Pairing is not always symmetrical. West US 3 is paired with East US, but East US is paired with West US.
- A few pairs cross geographies. Brazil South pairs with South Central US.
- Some secondary regions are **restricted access**, intended for disaster recovery only (for example France South, Germany North, Norway West, Switzerland West). You need to request access.

**Newer regions are not paired.** Regions such as Italy North, Poland Central, Spain Central, Israel Central, Qatar Central, Mexico Central, New Zealand North, Indonesia Central, Malaysia West, Austria East, Belgium Central, Chile Central and Denmark East have three availability zones and no pair. Microsoft's guidance has shifted accordingly: use availability zones for high availability, and choose your own second region for disaster recovery based on your requirements rather than relying on the pair.

### Data residency

Microsoft commits that customer data at rest for a **regional** service stays within the geography of the region you chose, with a few documented exceptions. That commitment has limits you must understand:

- **Non-regional services** (Entra ID, Front Door, Traffic Manager, and others) may store or process data globally or per a separately documented boundary.
- **Geo-redundant options** replicate to another region, usually in the same geography but not always.
- **AI model deployments** have their own rule, below.

The **EU Data Boundary** is a separate commitment to store and process customer data for core services within the EU and EFTA.

### AI and the region question

Foundry model deployments add a second axis to region choice. The deployment type decides where inference may happen:

| Deployment type | Where prompts are processed | Data at rest |
|---|---|---|
| **Global** | Any Azure region worldwide with capacity | In the resource's geography |
| **Data Zone** | Any region within a defined zone (for example the EU or the US) | In the resource's geography |
| **Regional** | Only the region of the resource | In the resource's geography |

Global deployments get the newest models first, the highest quota and the lowest price. Regional deployments give the tightest residency story but fewer models and less capacity. A client with an EU-only processing requirement can use Data Zone or Regional, not Global. Topic 15 covers this fully.

### Sovereign and special clouds

Separate instances of Azure with separate identity, endpoints and operations:

- **Azure Government:** for US government agencies and their partners, operated by screened US persons.
- **Azure operated by 21Vianet:** Azure in China, run by a local operator.

Service availability and feature parity lag the public cloud, often noticeably for new AI features. Topic 03 covers sovereign options further.

### Edge presence

Beyond regions, Microsoft runs a global network with many edge points of presence used by Front Door, CDN and the ExpressRoute and Microsoft peering edge. These are where user traffic enters Microsoft's backbone. You don't deploy resources to them.

## How it works on Azure

### Choosing a region

Work through these in order. The first two can eliminate options outright.

1. **Compliance and residency.** Where is the data legally allowed to be stored and processed?
2. **Service and feature availability.** Does the region offer every service, SKU and model you need? Check *Products available by region* and, for AI, the model availability tables.
3. **Availability zones.** Does the region have them, and do your services support them there?
4. **Latency.** How close is it to users and to systems you integrate with?
5. **Capacity and quota.** Can you actually get the VM family, GPUs or model throughput you need? Popular regions are sometimes constrained.
6. **Cost.** Prices differ by region, sometimes by a wide margin.
7. **Disaster recovery partner.** If you need a second region, which one, and does it meet points 1 and 2?

### Redundancy ladder

| Level | Protects against | Typical mechanism |
|---|---|---|
| Single instance | Nothing beyond hardware self-healing | One VM with premium disks |
| Multiple instances, one zone | Host and rack failure | Availability set, scale set |
| Zone-redundant | Data centre or zone failure | Spread across three zones |
| Multi-region | Region-wide failure | Second region plus global routing |

Each step up costs more and adds complexity. Stop at the level the business requirement justifies.

## Design decisions and trade-offs

- **Zones before regions.** Zone redundancy gives strong availability with synchronous replication and little added complexity. Multi-region introduces asynchronous replication, data-loss windows and failover orchestration. Go multi-region only when recovery objectives or regulation demand it.
- **Paired vs chosen secondary.** Use the pair when you depend on a service that only replicates to it (GRS, some backup features). Otherwise choose on latency, capacity and compliance.
- **Keep dependent resources together.** An app in one region calling a database in another pays latency and egress on every request, and now depends on two regions being up.
- **Residency vs capability for AI.** Global deployments are cheaper and better stocked. Be explicit with the client about whether in-geography *processing* is truly required, or only in-geography *storage*.
- **Separate the Foundry region from the app region if you must.** When the model you need is not in your primary region, it is common to host the model deployment in another region and call it over the backbone. Account for the latency and residency implications.

## Hands-on

Requires the Azure CLI and any subscription.

```bash
# List all regions available to your subscription
az account list-locations \
  --query "sort_by([?metadata.regionType=='Physical'], &name)[].{Name:name, Display:displayName, Geography:metadata.geographyGroup, Pair:metadata.pairedRegion[0].name}" \
  -o table

# Which regions have availability zones?
az account list-locations \
  --query "[?availabilityZoneMappings != null].{Region:name, Zones:length(availabilityZoneMappings)}" \
  -o table

# See how your logical zones map to physical zones in one region
az account list-locations \
  --query "[?name=='swedencentral'].availabilityZoneMappings[]" -o table

# Which VM sizes are available in a region, and in which zones?
az vm list-skus --location swedencentral --resource-type virtualMachines \
  --query "[?starts_with(name,'Standard_D4s')].{Size:name, Zones:locationInfo[0].zones}" -o table
```

Then answer for your own nearest region: how many zones does it have, is it paired, and with what?

## Security, cost and reliability notes

- **Security:** use Azure Policy's *Allowed locations* to stop resources being created outside approved regions. It is the simplest residency control there is.
- **Cost:** cross-region and cross-zone data transfer is billed. Chatty cross-region architectures generate real egress cost. Region price differences can exceed 20 percent for the same SKU.
- **Reliability:** zone-redundant SKUs sometimes cost more or need a higher tier. Budget for them at design time. Test failover; an untested DR region is a hope, not a plan.

## Common mistakes

- Choosing a region for latency alone and then finding the required model or SKU is unavailable there.
- Assuming every region has a pair.
- Assuming zone 1 is the same physical zone across subscriptions.
- Using GRS for "disaster recovery" without realising failover semantics and that the secondary is fixed.
- Deploying a "zone-redundant" architecture where one component (often a single-instance VM or a non-zonal SKU) is a single point of failure.
- Selecting a Global model deployment for a workload with a strict in-region processing requirement.

## Check yourself

1. Define region, availability zone and geography.
2. What is the difference between a zonal and a zone-redundant resource?
3. Give two benefits of region pairs.
4. Why can't you assume a newer region has a paired region, and what should you do instead?
5. List five factors in choosing a region.
6. A client requires that prompts are processed only inside the EU. Which Foundry deployment types are acceptable?
7. Why might zone 2 in subscription A not be the same data centre as zone 2 in subscription B?

<details>
<summary>Answers</summary>

1. A region is a set of data centres within a latency-defined perimeter where you deploy resources. An availability zone is a physically separate location within a region with independent power, cooling and networking. A geography is a market containing one or more regions that forms a data residency boundary.
2. A zonal resource is pinned to one zone you choose. A zone-redundant resource is automatically spread or replicated across zones by the platform.
3. Sequenced platform updates, prioritised recovery, and a defined replication target for geo-redundant services.
4. Newer regions are launched with three zones and no pair. Use zones for high availability and select a secondary region yourself for disaster recovery.
5. Compliance and residency, service availability, zone support, latency, capacity and quota, cost, DR partner.
6. Data Zone (EU) and Regional. Not Global.
7. Zone numbers are logical identifiers mapped per subscription to physical zones.

</details>

## References

- [List of Azure regions](https://learn.microsoft.com/azure/reliability/regions-list)
- [What are Azure regions?](https://learn.microsoft.com/azure/reliability/regions-overview)
- [What are availability zones?](https://learn.microsoft.com/azure/reliability/availability-zones-overview)
- [Azure paired regions](https://learn.microsoft.com/azure/reliability/regions-paired)
- [Select Azure regions (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-setup-guide/regions)
- [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/)
- [Data residency in Azure](https://azure.microsoft.com/explore/global-infrastructure/data-residency/)
- [Deployment types for Foundry Models](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
