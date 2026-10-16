# Spot, Reserved Capacity and Compute Fleet

> Topic: 06 · Compute · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

The same VM can cost very different amounts depending on how you buy it, and whether it is actually available when you ask is a separate question again. This file untangles the pricing models (pay-as-you-go, reservations, savings plans, Spot) from the capacity guarantee (capacity reservations), and shows how to combine them. The logic transfers directly to AI: GPU capacity follows the same rules, and provisioned throughput for models is the same commit-for-discount idea.

## Key concepts

### Three separate questions

| Question | Answered by |
|---|---|
| **What rate do I pay?** | Pay-as-you-go, reservation, savings plan, Spot, hybrid benefit |
| **Will capacity be there when I ask?** | On-demand capacity reservation (or already-running instances) |
| **Am I allowed to ask?** | Quota |

People routinely confuse the first two. A **reservation** (reserved instance) is a billing discount. A **capacity reservation** is a guarantee of hardware. They are different products that work well together.

### Pay-as-you-go

Per-second billing at list price, no commitment. The baseline everything else is compared with. Right for unpredictable, short-lived or new workloads until usage stabilises.

### Reservations (reserved VM instances)

Commit to a **VM family in a region** for **1 or 3 years** for a discount of up to roughly 72%.

| Property | Detail |
|---|---|
| What it is | A **billing discount** automatically applied to matching running VMs each hour |
| Scope | Shared (across the billing account), management group, subscription or resource group |
| **Instance size flexibility** | The discount applies across sizes in the same series group by ratio, so a reservation for one larger size can cover several smaller ones |
| Payment | Upfront or monthly, same total |
| Use it or lose it | Unused reserved hours are not carried forward |
| Changes | Exchanges and refunds are possible within Microsoft's policy and limits, which has been tightened over time. Check current terms before relying on them |
| Covers | Compute only. Not the OS licence (use Azure Hybrid Benefit), disks or networking |
| Capacity | Not guaranteed by a reservation scoped for flexibility. An option to prioritise capacity exists for single-size, single-subscription scope, but it is not an SLA |

Best for steady, predictable VMs you are confident will run in that family and region for the term.

Reservations exist for many other services too: SQL Database, Cosmos DB, Storage, App Service, Databricks, Fabric capacity, and **provisioned throughput units** for Foundry Models.

### Savings plan for compute

Commit to an **hourly spend** (for example 5.00 per hour) for **1 or 3 years**, for a discount of up to roughly 65%.

| Property | Detail |
|---|---|
| What it is | Usage up to the committed hourly amount is charged at savings plan rates; usage above it at pay-as-you-go |
| **Flexibility** | Applies automatically across VM families, sizes, **regions**, and operating systems, and across compute services: VMs, Dedicated Hosts, Container Instances, App Service Premium v3 and Isolated v2, Functions Premium, Container Apps, and others |
| Application order | To whichever eligible usage gives the greatest discount first |
| Use it or lose it | Committed hours not consumed are lost |
| Changes | Cannot be cancelled or exchanged once bought |

Best when compute spend is steady in total but the mix of sizes, regions or services changes.

### Reservation or savings plan?

| | Reservation | Savings plan |
|---|---|---|
| Discount | Higher | Slightly lower |
| Flexibility | Family and region fixed | Any family, region, eligible service |
| Commit to | A resource type | A spend amount |
| Good for | Stable, known workloads | Evolving estates, modernisation, mixed services |

They combine. Azure applies **reservations first**, then savings plans, then pay-as-you-go. A common strategy: reservations for the stable core you are sure about, a savings plan for the predictable-but-shifting layer above it, pay-as-you-go and Spot for the rest.

Azure Advisor and Cost Management recommend purchases from your usage history. Buy in increments and review regularly, since over-commitment is money spent on nothing.

### Azure Hybrid Benefit

Use existing **Windows Server** and **SQL Server** licences with Software Assurance (or qualifying subscriptions), and **RHEL** and **SLES** subscriptions, on Azure. You pay only the base compute rate. It stacks with reservations and savings plans, and for Windows Server and SQL Server workloads it is frequently the largest single saving.

### Spot virtual machines

Unused Azure capacity at discounts of up to roughly 90%, with the condition that Azure can take it back.

| Property | Detail |
|---|---|
| **Eviction** | When Azure needs the capacity, or when the Spot price rises above your maximum |
| **Notice** | About **30 seconds**, through Scheduled Events |
| **Eviction policy** | *Deallocate* (keep disks, pay for storage, restart later if capacity returns) or *Delete* (everything removed) |
| **Max price** | A cap you set, or `-1` meaning "up to the pay-as-you-go price": never evicted for price, only for capacity |
| **SLA** | None |
| **Quota** | Separate Spot vCPU quota |
| **Price** | Varies by size, region and time. Eviction rates and price history are shown in the portal and available through Resource Graph |

**Good for:** batch and queue-driven processing, CI/CD agents, rendering, large-scale testing, stateless scale-out tiers above a guaranteed base, and **model training and fine-tuning that checkpoints**.

**Not for:** anything that cannot tolerate interruption: single-instance services, databases without replication, latency-critical user-facing tiers running entirely on Spot.

**Designing for eviction**

- **Checkpoint** progress regularly to durable storage.
- **Handle the Scheduled Event**: stop taking work, flush state, deregister.
- Make work **idempotent** and re-queueable.
- **Diversify**: several sizes and zones lower the chance that all your capacity is reclaimed at once.
- Keep an **on-demand base** for the minimum acceptable capacity.

**Spot in scale sets**

- **Spot priority mix** (Flexible orchestration): for example a base of 4 regular instances, then 25% regular and 75% Spot above that.
- **Try to restore**: the platform attempts to bring back evicted instances when capacity returns.
- **Instance mix** with allocation strategies (*lowest price*, *capacity optimised*, *prioritised*).

In AKS, Spot node pools carry a taint so that only workloads that tolerate interruption land there.

### On-demand capacity reservations

Reserve **compute capacity** for a specific VM size in a region or zone, for as long as you want, with no term commitment.

| Property | Detail |
|---|---|
| What it guarantees | That the reserved quantity of that size is available to you, with an SLA |
| Billing | At the **pay-as-you-go** rate for the size, **whether or not** VMs are running in it. Reservations (reserved instances) and savings plans can discount it |
| Structure | A **capacity reservation group** holds reservations for sizes and quantities; VMs and scale sets are associated with the group to consume it |
| Flexibility | Create and delete at any time; change the quantity |
| Sharing | Can be shared across subscriptions |
| Overallocation | You can run more VMs than reserved; the extras are ordinary on-demand allocations without a guarantee |

Use it for:

- **Disaster recovery**: guarantee the secondary region can actually host you during a regional event, when everyone else is failing over too.
- **Planned events and launches**: secure capacity ahead of a known peak.
- **Scarce sizes**: GPUs and other constrained SKUs, where "quota granted" does not mean "capacity available".
- **Zone-loss headroom** for critical tiers.

Combining a capacity reservation (guarantee) with a reserved instance (discount) gives guaranteed capacity at a reduced rate.

### Azure Compute Fleet

A service for launching and managing a large, mixed pool of VMs with one request.

| Capability | Detail |
|---|---|
| Scale | Up to 10,000 VMs per fleet |
| Mix | **Spot and standard** together, with target capacities for each |
| Diversity | Many VM sizes, specified explicitly or by **attribute-based selection** (give vCPU and memory ranges and let the service pick) |
| Allocation strategies | Spot: *price-capacity optimised*, *capacity optimised*, *lowest price*. Standard: *lowest price*, *prioritised* |
| Regions | Multi-region fleets |
| Maintenance | *Maintain* mode replaces evicted or failed instances to hold the target; *request* mode is one-off |

It suits very large, flexible workloads where you care about total vCPUs more than specific instance types: big batch runs, simulation, large-scale data processing, distributed training. For an ordinary application tier, a scale set with instance mix is simpler.

### Quota in the picture

All of the above operate inside your **vCPU quota** per family per region (and a separate total regional and Spot quota). A purchase of reservations or a capacity reservation does not raise quota; request increases separately through the *Quotas* blade, and do it early for GPU families.

**Quota groups** let you pool and move quota between subscriptions in a management group, which helps large organisations avoid stranded allocations.

## How it works on Azure

### A layered purchasing model

```text
                       ┌────────────────────────────────────────┐
 Variable / interruptible │  Spot                                  │  batch, CI, training, burst
                       ├────────────────────────────────────────┤
 Unpredictable         │  Pay-as-you-go                         │  new workloads, spikes
                       ├────────────────────────────────────────┤
 Predictable, shifting │  Savings plan for compute              │  total spend steady, mix changes
                       ├────────────────────────────────────────┤
 Stable core           │  Reservations (+ Azure Hybrid Benefit) │  always-on, known family and region
                       └────────────────────────────────────────┘
 Guarantee where needed:  on-demand capacity reservation (DR, launches, GPUs)
```

### Finding what to buy

1. **Right-size first.** Committing to oversized VMs locks in waste.
2. Review **Advisor** and **Cost Management** reservation and savings plan recommendations, which model different commitment levels against the last 7, 30 or 60 days.
3. Cover the **stable baseline** only. A good target is high utilisation of the commitment (above 95%) rather than maximum coverage.
4. Choose **scope**: shared scope maximises utilisation across subscriptions; narrower scope gives a team its own benefit.
5. **Monitor utilisation** and set alerts for under-use and upcoming expiry.

### The AI parallel

| VM concept | AI model equivalent |
|---|---|
| Pay-as-you-go | Standard deployments, billed per token |
| Reservation | **Provisioned throughput reservation**: a term commitment discounting PTUs |
| Capacity guarantee | **Provisioned throughput units** themselves: dedicated model processing capacity |
| Spot | **Batch** deployments: cheaper, asynchronous, no latency guarantee |
| Quota | Tokens-per-minute and PTU quota per model, region and deployment type |
| Savings plan | Commitment constructs for Foundry and Azure consumption, where offered |

The reasoning is identical: find the steady baseline, commit to that, and serve the variable part on demand. Topic 15 and topic 29 apply it to models.

For GPU VMs specifically: quota is often zero by default, capacity is regionally constrained, Spot GPU capacity is volatile, and reservations make sense only for training or inference clusters that run continuously. Capacity reservations are worth considering for production inference on scarce sizes.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| When to commit | After usage has stabilised for a month or two |
| 1 year or 3 | 3-year discounts are larger, with more risk that technology or needs change. Mix terms |
| Reservation or savings plan | Reservations for what will not move; a savings plan for everything else steady |
| How much to cover | The baseline you are sure of. Under-commit slightly; unused commitment is pure loss |
| Spot share | As high as the workload's tolerance for interruption allows, above an on-demand floor |
| Capacity reservation | Where being unable to deploy is unacceptable. It bills continuously, so size it to the real requirement |
| Who buys | Centrally (FinOps), with shared scope, so discounts are not stranded in one team's subscription |

## Hands-on

```bash
LOC=swedencentral; RG=rg-spot-lab
az group create -n $RG -l $LOC -o none

# 1. Quota: regular and Spot
az vm list-usage -l $LOC --query "[?contains(name.value,'lowPriorityCores') || name.value=='cores' || contains(name.value,'standardDASv6Family')].{Name:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# 2. Compare pay-as-you-go and Spot prices for a size (public Retail Prices API, no auth)
curl -s "https://prices.azure.com/api/retail/prices?\$filter=armRegionName eq '$LOC' and armSkuName eq 'Standard_D4as_v6' and priceType eq 'Consumption'" \
  | jq -r '.Items[] | select(.productName|test("Windows")|not) | [.skuName, .unitPrice, .unitOfMeasure] | @tsv'

# 3. Reservation and savings plan prices for the same size
curl -s "https://prices.azure.com/api/retail/prices?\$filter=armRegionName eq '$LOC' and armSkuName eq 'Standard_D4as_v6' and priceType eq 'Reservation'" \
  | jq -r '.Items[] | [.reservationTerm, .unitPrice] | @tsv'

# 4. A Spot VM that is evicted only for capacity
az vm create -g $RG -n vm-spot --image Ubuntu2404 --size Standard_D2as_v6 \
  --priority Spot --max-price -1 --eviction-policy Deallocate \
  --public-ip-address "" --admin-username azureuser --generate-ssh-keys -o none
az vm show -g $RG -n vm-spot --query "{priority:priority, evictionPolicy:evictionPolicy, maxPrice:billingProfile.maxPrice}"

# 5. Simulate an eviction and watch the Scheduled Event
az vm run-command invoke -g $RG -n vm-spot --command-id RunShellScript --no-wait \
  --scripts 'for i in $(seq 1 30); do curl -s -H Metadata:true "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01" >> /tmp/events.log; echo >> /tmp/events.log; sleep 2; done'
az vm simulate-eviction -g $RG -n vm-spot
sleep 90
az vm get-instance-view -g $RG -n vm-spot --query "instanceView.statuses[1].displayStatus"

# 6. A capacity reservation (bills at pay-as-you-go while it exists: delete promptly)
az capacity reservation group create -g $RG -n crg-lab -l $LOC --zones 1 -o none
az capacity reservation create -g $RG --capacity-reservation-group crg-lab -n cr-d2 \
  --sku Standard_D2as_v6 --capacity 1 --zone 1 -o none
az capacity reservation show -g $RG --capacity-reservation-group crg-lab -n cr-d2 --instance-view \
  --query "{reserved:sku.capacity, used:length(instanceView.utilizationInfo.virtualMachinesAllocated)}"

az group delete -n $RG --yes --no-wait
```

Then in the portal: **Advisor → Cost** for reservation and savings plan recommendations, and **Reservations → Add** to see how the purchase recommendation models coverage (do not complete a purchase).

Exercise: an estate runs 40 `D4as_v6` VMs continuously, a further 10 to 60 depending on time of day, and a nightly batch job of 200 vCPUs for three hours that can be retried. Propose the purchasing mix.

<details>
<summary>Suggested answer</summary>

Reservations (3-year, shared scope) for the 40 always-on VMs, with Azure Hybrid Benefit if Windows. A savings plan sized to the reliably present part of the variable tier (the 10 that always run). Pay-as-you-go for the remaining burst up to 60. Spot, with several sizes and checkpointing, for the nightly batch. A capacity reservation only if failing to scale at peak is unacceptable.

</details>

## Security, cost and reliability notes

- **Cost:** commitments are liabilities. Track utilisation monthly; an unused reservation or savings plan is worse than pay-as-you-go.
- **Cost:** capacity reservations bill continuously even when empty. Automate their creation and removal around events.
- **Cost:** Spot with `Deallocate` still pays for disks while evicted.
- **Reliability:** Spot has no SLA. Never let total capacity depend on it.
- **Reliability:** quota and discounts do not guarantee capacity. For anything that must scale or fail over, test it or reserve it.
- **Security:** who can purchase reservations is a financial control. Restrict the reservation purchaser role and review purchases.

## Common mistakes

- Believing a reserved instance guarantees capacity.
- Buying commitments before right-sizing.
- Committing to 100% of current usage.
- Reservations scoped to one subscription, unused while another pays full price.
- Production single instances on Spot.
- No eviction handling: a 30-second notice ignored.
- One VM size for a large Spot pool.
- A capacity reservation left running after the event.
- Forgetting Azure Hybrid Benefit on Windows and SQL workloads.
- Discovering the disaster recovery region has no capacity during the disaster.

## Check yourself

1. What is the difference between a reserved instance and a capacity reservation?
2. What is each of a reservation and a savings plan a commitment to?
3. In what order are reservations, savings plans and pay-as-you-go applied?
4. When can a Spot VM be evicted, and how much notice is given?
5. What does `--max-price -1` mean?
6. Name three workloads suited to Spot and two that are not.
7. Why create a capacity reservation in a disaster recovery region?
8. How do these concepts map to Foundry model deployments?

<details>
<summary>Answers</summary>

1. A reserved instance is a billing discount for a term commitment. A capacity reservation guarantees that hardware of a given size is available to you, billed at pay-as-you-go whether used or not.
2. A reservation commits to a resource type (a VM family in a region). A savings plan commits to an hourly spend amount across eligible compute.
3. Reservations first, then savings plans, then pay-as-you-go.
4. When Azure needs the capacity back or the Spot price exceeds your maximum price; about 30 seconds' notice through Scheduled Events.
5. You will pay up to the pay-as-you-go price, so the VM is evicted only for capacity, never for price.
6. Suited: batch processing, CI/CD agents, checkpointed model training, stateless burst capacity. Not suited: single-instance services, databases without replication, latency-critical tiers running only on Spot.
7. In a regional outage many customers fail over at once and capacity may be unavailable; the reservation guarantees you can deploy.
8. Standard deployments are pay-as-you-go; provisioned throughput units are dedicated capacity; PTU reservations are the term discount; batch deployments resemble Spot's cheaper, less immediate capacity; quota applies per model and region.

</details>

## References

- [What are Azure Reservations?](https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations)
- [Virtual machine size flexibility with reserved VM instances](https://learn.microsoft.com/azure/virtual-machines/reserved-vm-instance-size-flexibility)
- [What is Azure savings plans for compute?](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview)
- [Decide between a savings plan and a reservation](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/decide-between-savings-plan-reservation)
- [Use Azure Spot Virtual Machines](https://learn.microsoft.com/azure/virtual-machines/spot-vms)
- [Spot priority mix](https://learn.microsoft.com/azure/virtual-machine-scale-sets/spot-priority-mix)
- [On-demand capacity reservation](https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview)
- [What is Azure Compute Fleet?](https://learn.microsoft.com/azure/azure-compute-fleet/overview)
- [Azure Hybrid Benefit](https://learn.microsoft.com/azure/virtual-machines/windows/hybrid-use-benefit-licensing)
- [Azure Retail Prices REST API](https://learn.microsoft.com/rest/api/cost-management/retail-prices/azure-retail-prices)
