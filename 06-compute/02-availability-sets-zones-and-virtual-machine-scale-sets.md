# Availability Sets, Zones and Virtual Machine Scale Sets

> Topic: 06 · Compute · Tracks: E ○ A ● C – · Status checked: 2026-10

## Why it matters

A single virtual machine fails when its host fails, is rebooted for maintenance, and cannot grow with demand. Availability and scale on IaaS come from running several instances and spreading them across failure boundaries. The constructs in this file are also what AKS node pools, Batch pools and virtual desktops are built on, so they explain behaviour you will see far beyond plain VMs.

## Key concepts

### Failure boundaries

| Boundary | Protects against |
|---|---|
| **Fault domain** | A rack: shared power and network switch |
| **Update domain** | A group of hosts rebooted together during planned maintenance |
| **Availability zone** | A data centre (or group) with independent power, cooling and network |
| **Region** | A whole metropolitan area |

### VM SLAs

Financially backed connectivity commitments (confirm current values in the SLA document):

| Configuration | SLA |
|---|---|
| Single VM, Premium SSD or Ultra disks on all disks | 99.9% |
| Single VM, Standard SSD | 99.5% |
| Single VM, Standard HDD | 95% |
| Two or more VMs in an **availability set** | 99.95% |
| Two or more VMs across **availability zones** | 99.99% |

The jump from 99.9% to 99.99% is from about 44 minutes of permitted downtime a month to about 4.

### Availability sets

A logical grouping that spreads VMs across **fault domains** (up to 3) and **update domains** (up to 20) **within one data centre**.

- Protects against rack failure and planned host updates.
- Does **not** protect against a data centre or zone failure.
- Free.
- VMs must be placed in the set at creation.

Availability sets are the legacy option. Use them only in regions without zones, or for software that explicitly requires them. For new designs choose zones.

### Availability zones

Each zone-enabled region has at least three zones. For VMs:

- A **zonal** VM is pinned to a zone you pick; its managed disks and any zonal public IP live there too.
- You get resilience by deploying instances in **two or three zones** behind a **zone-redundant** load balancer.
- Zone numbers are **logical per subscription** (topic 01, file 03).
- Latency between zones is low (designed for under 2 ms round trip), suitable for synchronous replication.
- Inter-zone data transfer is billed.

Constraints: not every size is available in every zone; a zonal VM cannot be moved to another zone without redeployment (a zone-move capability exists for some configurations); and zonal Premium SSD v2 and Ultra disks tie the VM to their zone.

### Virtual Machine Scale Sets

A scale set manages a group of VMs as one resource: creating, updating, scaling and distributing them.

**Orchestration modes**

| | **Flexible** (default, recommended) | **Uniform** (older) |
|---|---|---|
| Instances | Standard VMs you can manage individually | Scale-set-specific instances from one model |
| Mixing sizes and priorities | Yes: several sizes, Spot and on-demand together | No |
| Scale | Up to 1,000 instances | Up to 1,000 (100 with custom images in some cases) |
| Spreading | Across zones, or across fault domains in a region | Across zones or fault domains |
| Add existing VMs | Yes | No |
| Use for | Almost everything new | Existing deployments; a few platform features that still need it |

Flexible orchestration combines the availability of zones or fault domains with standard VM APIs. Microsoft directs all new work to it.

**What a scale set gives you**

| Capability | Detail |
|---|---|
| **Autoscale** | Add or remove instances on metrics, schedules or predictions |
| **Zone balancing** | Spreads instances evenly across the zones you select; rebalances after failures |
| **Automatic instance repairs** | Replaces instances reported unhealthy by the Application Health extension or a load balancer probe |
| **Upgrade policies** | *Manual*, *Automatic* or *Rolling* application of a changed model (image, size, extensions) |
| **Automatic OS image upgrades** | Rolls out new platform or gallery image versions in batches with health checks and rollback |
| **Instance mix** | Several VM sizes with an allocation strategy, improving capacity and Spot outcomes |
| **Spot priority mix** | A base of on-demand instances plus a percentage split of Spot above it |
| **Scale-in policy** | Which instances are removed first: default (balance across zones, then highest ID), newest, or oldest |
| **Instance protection** | Shield chosen instances from scale-in or from scale set actions |
| **Terminate notification** | A Scheduled Event giving up to 15 minutes to drain before deletion |
| **Standby pools** | Pre-provisioned, optionally deallocated instances to cut scale-out time |
| **Resilient create and delete** | Automatic retries of transient provisioning failures |
| **Proximity placement, capacity reservation, dedicated host association** | Placement controls |

### Autoscale

Configured in Azure Monitor against the scale set.

| Rule type | Example |
|---|---|
| **Metric-based** | Average CPU > 70% for 10 minutes → add 2; < 30% for 15 minutes → remove 1 |
| **Custom metric** | Queue length, requests per instance, an application metric from Application Insights |
| **Schedule** | Weekdays 08:00–18:00 minimum 6; otherwise minimum 2 |
| **Predictive** | Machine learning on CPU history scales out ahead of recurring peaks |

Design points:

- Set **minimum, maximum and default** instance counts. The minimum is your availability floor; with three zones, at least 3 (or 2 with a plan for losing a zone).
- **Scale out fast, scale in slowly.** Aggressive scale-in causes flapping.
- Pair every scale-out rule with a scale-in rule, with a gap between thresholds.
- Mind the **cooldown** so new instances can take load before the next decision.
- Scale on a metric that reflects **user-visible load** (queue depth, requests) where CPU does not.
- Instances must be **stateless** or externalise state. Autoscale removes instances without ceremony.
- New instances take minutes to provision and warm up. Faster options: smaller images, ephemeral OS disks, standby pools, predictive autoscale.

### Zone-redundant, zonal and regional placement

| Placement | Behaviour | Use |
|---|---|---|
| **Zone-spanning scale set** (zones 1, 2, 3) | Instances spread and balanced across zones | The default for resilience |
| **Zonal** (one zone) | All instances in one zone | Latency-sensitive tiers that must be co-located, paired with another zonal set elsewhere |
| **Regional** (no zones) | Spread across fault domains | Regions without zones; capacity flexibility |

**Zone balance vs capacity.** Strict zone balancing can block scale-out when one zone lacks capacity for your size. Best-effort balancing and **instance mix** (several sizes) make allocation more likely to succeed.

### Load balancing instances

Scale sets integrate with Load Balancer and Application Gateway backend pools: instances are added and removed automatically. Health probes plus the Application Health extension drive both traffic routing and instance repair.

## How it works on Azure

### A resilient VM tier

```text
Zone-redundant Standard Load Balancer / Application Gateway
        │
Virtual Machine Scale Set (Flexible), zones 1·2·3
   ├─ instance mix: D4as_v6, D4s_v6, D4ads_v6      (capacity flexibility)
   ├─ autoscale: min 3, max 30, on requests per instance; scheduled floor in business hours
   ├─ rolling upgrades, 20% batches, health-gated
   ├─ automatic instance repairs via Application Health extension
   ├─ ephemeral OS disks, image from Compute Gallery
   └─ terminate notification → drain connections
State lives outside: managed database, storage, cache
```

### Surviving a zone failure

With instances spread over three zones, losing one removes a third of capacity. Either:

- **Over-provision:** run at 150% of peak need so two zones can carry the load; or
- **Scale on failure:** rely on autoscale to add instances in surviving zones, accepting a period of degraded performance and the risk that everyone else in the region is trying to do the same.

For critical workloads, over-provision or hold a capacity reservation. Stateful tiers (databases, queues) must themselves be zone-redundant, or the compute tier's resilience is irrelevant.

### Updates without downtime

| Change | Approach |
|---|---|
| New application version | New image version in the gallery, then a **rolling upgrade** in health-gated batches; or blue-green with a second scale set and a traffic switch |
| OS patches | **Automatic OS image upgrades** for image-based fleets; Update Manager or automatic guest patching for longer-lived VMs |
| Size change | Update the model and roll |
| Rollback | Re-roll to the previous image version. Keep it available |

Set **max surge** so new instances are created before old ones are removed, preserving capacity during the roll.

### Where you meet scale sets indirectly

| Service | Built on |
|---|---|
| AKS node pools | Scale sets (Uniform or Flexible, depending on configuration) or standalone VMs |
| Azure Batch pools | Scale sets |
| Azure Virtual Desktop session hosts | VMs, often managed with scale-set-like tooling |
| Service Fabric, Azure Spring Apps, Databricks clusters | Scale sets |

Understanding upgrade policies, zone balancing and Spot behaviour explains what those services do on your behalf.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Availability set or zones | Zones, wherever available |
| Flexible or Uniform | Flexible |
| Two or three zones | Three: a zone loss costs a third of capacity, not half |
| Single size or instance mix | Instance mix improves the chance of getting capacity, particularly with Spot. Sizes should be similar in CPU and memory |
| Scale metric | The one that tracks user-facing load |
| Minimum instances | Enough to survive a zone loss at acceptable performance |
| Stateful VMs | Scale sets work best stateless. For clustered stateful software, use zones with the product's own replication, or a managed service |
| VMs at all | Container Apps, App Service and AKS provide these behaviours with less to manage (file 12) |

## Hands-on

```bash
RG=rg-vmss-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

cat > cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages: [nginx]
runcmd:
  - echo "<h1>$(hostname) zone $(curl -s -H Metadata:true 'http://169.254.169.254/metadata/instance/compute/zone?api-version=2021-02-01&format=text')</h1>" > /var/www/html/index.html
EOF

# 1. A Flexible scale set across three zones behind a zone-redundant load balancer
az vmss create -g $RG -n vmss-lab --orchestration-mode Flexible \
  --image Ubuntu2404 --vm-sku Standard_D2as_v6 --instance-count 3 --zones 1 2 3 \
  --lb-sku Standard --upgrade-policy-mode Rolling --custom-data cloud-init.yaml \
  --admin-username azureuser --generate-ssh-keys -o none
# (the generated VNet needs an outbound path for package install; the Standard LB's outbound rule provides one)

# 2. Where did the instances land?
az vm list -g $RG --query "[].{Name:name, Zone:zones[0], Size:hardwareProfile.vmSize}" -o table

# 3. Autoscale: 3 to 9 instances on CPU, with a scheduled floor
VMSS_ID=$(az vmss show -g $RG -n vmss-lab --query id -o tsv)
az monitor autoscale create -g $RG -n as-vmss-lab --resource $VMSS_ID --min-count 3 --max-count 9 --count 3 -o none
az monitor autoscale rule create -g $RG --autoscale-name as-vmss-lab --scale out 2 \
  --condition "Percentage CPU > 70 avg 5m" --cooldown 5 -o none
az monitor autoscale rule create -g $RG --autoscale-name as-vmss-lab --scale in 1 \
  --condition "Percentage CPU < 30 avg 10m" --cooldown 10 -o none

# 4. Generate load on every instance and watch it scale
for VM in $(az vm list -g $RG --query "[].name" -o tsv); do
  az vm run-command invoke -g $RG -n $VM --command-id RunShellScript \
    --scripts "nohup sh -c 'timeout 900 yes > /dev/null' >/dev/null 2>&1 & nohup sh -c 'timeout 900 yes > /dev/null' >/dev/null 2>&1 &" --no-wait
done
sleep 600
az vm list -g $RG --query "[].{Name:name, Zone:zones[0]}" -o table
az monitor autoscale show -g $RG -n as-vmss-lab --query "profiles[0].capacity"
az monitor activity-log list -g $RG --offset 30m --query "[?contains(operationName.value,'autoscale')].{op:operationName.localizedValue, status:status.value}" -o table

# 5. Simulate a zone failure: delete every instance in zone 1 and watch the set rebalance
for VM in $(az vm list -g $RG --query "[?zones[0]=='1'].name" -o tsv); do az vm delete -g $RG -n $VM --yes --no-wait; done

az group delete -n $RG --yes --no-wait
```

Request the load balancer's public IP a few times and watch the host name and zone change. In the portal, look at the scale set's *Scaling* history, *Instances* by zone and *Upgrade policy*.

Paper exercise: a tier needs 12 instances at peak and must survive the loss of a zone with no performance loss. How many instances do you run, and what does that cost relative to no zone redundancy?

<details>
<summary>Answer</summary>

Two surviving zones must carry 12 instances, so 6 per zone and 18 in total: 50% more than the 12 needed without the requirement. The alternative is 12 across three zones and accepting two-thirds capacity during a zone outage while autoscale catches up.

</details>

## Security, cost and reliability notes

- **Security:** scale set instances are created from an image without human intervention. The image must already be hardened and patched; keep the gallery image current and roll it out automatically.
- **Security:** no public IPs on instances; one controlled entry point.
- **Cost:** autoscale is the main cost lever for variable load, alongside Spot for interruptible capacity. A minimum set too high wastes money; too low risks availability.
- **Cost:** zone redundancy with headroom costs more by design. State the trade-off explicitly.
- **Reliability:** an SLA applies only if the configuration meets its conditions: instances in at least two zones, behind a zone-redundant entry point.
- **Reliability:** test scale-out, scale-in and zone loss before production. Allocation failures and slow warm-up are discovered in tests or in incidents.

## Common mistakes

- A single VM described as "highly available" because its disk is Premium.
- Availability sets where zones were available.
- Zone-spread instances behind a zonal load balancer or database.
- A minimum of one instance.
- Scaling on CPU when the bottleneck is elsewhere.
- Scale-in as aggressive as scale-out, causing oscillation.
- State on instances that autoscale then deletes.
- No health signal, so broken instances keep receiving traffic.
- One VM size only, then allocation failures at scale-out.
- Manual upgrade policy, leaving the fleet on mixed, old images.
- Assuming capacity will be there after a zone outage without over-provisioning or a reservation.

## Check yourself

1. What does an availability set protect against, and what does it not?
2. What SLA applies to two VMs across availability zones?
3. Why is Flexible orchestration recommended?
4. Give three autoscale design rules.
5. What does automatic instance repair need in order to work?
6. How does instance mix help?
7. How do you design a tier to survive a zone loss without degradation?
8. Which other Azure services are built on scale sets?

<details>
<summary>Answers</summary>

1. Rack-level failures and planned host updates within one data centre. Not a data centre or zone failure.
2. 99.99%.
3. It uses standard VMs, mixes sizes and Spot with on-demand, spreads across zones, and is where new capabilities are added.
4. Any three of: set sensible minimum and maximum; scale out quickly and in slowly; leave a gap between thresholds; scale on a metric reflecting user load; keep instances stateless; allow for warm-up and cooldown.
5. A health signal: the Application Health extension or a load balancer health probe.
6. Allowing several VM sizes increases the chance that capacity is available at scale-out and improves Spot allocation and price.
7. Over-provision so the remaining zones can carry full load (for three zones, 150% of need), or hold reserved capacity, and make stateful dependencies zone-redundant.
8. AKS node pools, Azure Batch pools, and others such as Databricks clusters and Service Fabric.

</details>

## References

- [Availability options for Azure Virtual Machines](https://learn.microsoft.com/azure/virtual-machines/availability)
- [What are Virtual Machine Scale Sets?](https://learn.microsoft.com/azure/virtual-machine-scale-sets/overview)
- [Orchestration modes for Virtual Machine Scale Sets](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes)
- [Create a scale set that uses availability zones](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones)
- [Overview of autoscale with scale sets](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview)
- [Automatic instance repairs](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-instance-repairs)
- [Upgrade policy modes](https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-upgrade-policy)
- [Instance mix](https://learn.microsoft.com/azure/virtual-machine-scale-sets/instance-mix-overview)
- [Standby pools](https://learn.microsoft.com/azure/virtual-machine-scale-sets/standby-pools-overview)
- [SLA for Virtual Machines](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services)
