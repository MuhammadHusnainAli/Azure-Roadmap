# VNet Peering and Azure Virtual Network Manager

> Topic: 05 · Networking · Tracks: E ○ A ● C – · Status checked: 2026-10

## Why it matters

Workloads live in their own virtual networks, and something has to connect them. Peering is the basic link. Hub-and-spoke is the topology nearly every organisation builds from it. Azure Virtual Network Manager is how that topology, and the security rules that must apply everywhere, stay consistent once there are more than a handful of networks.

## Key concepts

### VNet peering

Peering connects two virtual networks so resources in each can communicate using private IPs over the Microsoft backbone.

| Property | Detail |
|---|---|
| Path | Microsoft backbone. No gateway, no public internet, no encryption overhead by default |
| Performance | Same as within a VNet; no bandwidth cap imposed by the peering |
| Scope | Same region (**regional peering**) or different regions (**global peering**) |
| Boundaries | Works across subscriptions and across Entra tenants |
| Requirement | **Non-overlapping** address spaces |
| Direction | A peering is two links, one from each side. Both must exist and show *Connected* |
| Cost | No hourly charge. Data transferred is billed per GB in both directions; global peering costs more than regional |

### Peering settings

Each side of the peering has four settings:

| Setting | Meaning |
|---|---|
| **Allow virtual network access** | Whether the peered VNet's range is included in the `VirtualNetwork` service tag, permitting traffic under default NSG rules |
| **Allow forwarded traffic** | Accept traffic that did not originate in the peered VNet, for example forwarded by a firewall in the hub. Needed on spokes for hub-routed traffic |
| **Allow gateway transit** | On the hub: let peered VNets use this VNet's VPN or ExpressRoute gateway |
| **Use remote gateways** | On the spoke: use the peer's gateway. The spoke cannot have its own gateway |

A standard hub-and-spoke peering:

| | Hub → Spoke link | Spoke → Hub link |
|---|---|---|
| Allow virtual network access | Yes | Yes |
| Allow forwarded traffic | Yes | Yes |
| Allow gateway transit | Yes | – |
| Use remote gateways | – | Yes |

### Non-transitivity

```text
Spoke A ◄──peering──► Hub ◄──peering──► Spoke B

Spoke A cannot reach Spoke B through the hub by peering alone.
```

To connect spokes you need either:

- a **routing device in the hub** (Azure Firewall, an NVA) plus user-defined routes on each spoke, which also gives inspection; or
- **direct peering** between the spokes (a mesh); or
- **Virtual WAN**, where the managed hub routes between connected VNets.

### Other peering facts

- **Address space changes** to a peered VNet are supported; the peering must be **synced** afterwards for the change to take effect.
- **Subnet peering** lets you peer selected subnets instead of entire address spaces, useful when parts of two VNets overlap or only some subnets should be exposed.
- **Global peering limitations** are few now, but some basic-tier or legacy resources are not reachable across it.
- **Limit:** 500 peerings per VNet by default, rising to 1,000 with Azure Virtual Network Manager connectivity configurations.
- Peering does **not** share DNS. Name resolution across peered VNets needs private DNS zones linked to both, or shared DNS servers.

### Peering vs other connection methods

| Method | Use for | Notes |
|---|---|---|
| **VNet peering** | VNet to VNet in Azure | Fastest, simplest, cheapest |
| **VNet-to-VNet VPN** | Rare cases needing IPsec encryption or transitive routing through a gateway | Throughput limited by gateway SKU; hourly gateway cost |
| **Virtual WAN** | Many VNets and branches with managed transit | File 13 |
| **Private Link service** | Exposing one service to another VNet or tenant without network-level connectivity; works with overlapping address spaces | File 05 |
| **ExpressRoute / VPN** | On-premises to Azure | File 12 |

### Hub-and-spoke topology

```text
                        On-premises
                             │  ExpressRoute / VPN
                    ┌────────┴────────┐
                    │       Hub       │  Firewall · Gateways · Bastion · DNS resolver
                    └─┬──────┬──────┬─┘
              peering │      │      │ peering
              ┌───────┴┐  ┌──┴────┐ ┌┴────────┐
              │ Spoke A│  │Spoke B│ │ Spoke C │   one per workload / environment
              └────────┘  └───────┘ └─────────┘
```

| Element | Holds |
|---|---|
| **Hub** | Shared network services: firewall, VPN and ExpressRoute gateways, Bastion, DNS Private Resolver, sometimes shared ingress |
| **Spoke** | One workload's resources, isolated from other spokes by default |

Benefits: shared services are paid for once, traffic is centrally inspected, workloads are isolated, and each spoke can sit in its own subscription with its own owners. One hub per region; hubs are peered or connected to each other for cross-region traffic.

### Azure Virtual Network Manager

AVNM is a management service for grouping virtual networks and applying **connectivity**, **security** and **routing** configurations to them at scale, across subscriptions and regions.

**Building blocks**

| Concept | Meaning |
|---|---|
| **Network manager** | The instance, with a **scope** (management groups and/or subscriptions) and enabled features |
| **Network group** | A set of VNets (or subnets). Membership is **static** (listed) or **dynamic** (defined by an Azure Policy condition, such as a tag or name pattern) |
| **Configuration** | A connectivity, security admin, routing or (where available) user-defined rule definition applied to network groups |
| **Deployment** | Committing a configuration to chosen **regions**. Nothing takes effect until deployed, which enables staged rollout |

Dynamic membership is what makes it scale: tag a new spoke `avnm-group=corp-spokes` at vending time and it joins the group and receives its peering, rules and routes automatically.

**Connectivity configurations**

| Topology | Result |
|---|---|
| **Hub and spoke** | Peers every VNet in the group to a chosen hub. Options: use the hub as gateway; enable **direct connectivity** between spokes in the group |
| **Mesh** | Every VNet in the group can reach every other, implemented as a *connected group* rather than individual peerings, so it scales far beyond the peering limit |

Connected groups give spoke-to-spoke connectivity without a hop through the hub and without creating N² peerings.

**Security admin rules**

Organisation-level rules evaluated **before** NSGs.

| Action | Behaviour |
|---|---|
| **Allow** | Traffic passes to NSG evaluation, which can still deny it |
| **Always Allow** | Traffic is allowed and NSGs are **not** evaluated. Use for traffic that must never be blocked, such as monitoring or domain controllers |
| **Deny** | Traffic is dropped. NSGs cannot override |

```text
Packet → Security admin rules (AVNM) → NSG rules (subnet, NIC) → destination
           deny: stop here
           always allow: skip NSGs
           allow / no match: continue
```

Typical use: deny inbound SSH, RDP and other high-risk ports from the internet across every VNet, so that no workload team can accidentally expose them, while leaving teams free to manage their own NSGs for everything else.

**Routing configurations (UDR management)**

Define routing intent once, for example "all spoke subnets send `0.0.0.0/0` to the hub firewall", and AVNM creates and maintains the route tables. Route tables it manages are updated as group membership changes.

**IP address management (IPAM)**

Create address **pools**, allocate non-overlapping ranges to VNets from them, reserve ranges for on-premises or other clouds, and see utilisation. It replaces the spreadsheet most organisations use.

**Virtual network verifier**

Analyses reachability between resources ("can this subnet reach that private endpoint on 443?") against the configured NSGs, admin rules, routes and peerings, without sending traffic.

### Pricing

AVNM is billed per subscription that contains a managed virtual network (the model changed from per-VNet), plus charges for some features. Check current pricing; for a large estate it is modest relative to the operational saving, but it is not free.

## How it works on Azure

### AVNM in a landing zone

```text
Network manager (scope: intermediate root management group)
├── Network group "corp-spokes"     dynamic: tag avnm=corp
├── Network group "online-spokes"   dynamic: tag avnm=online
├── Connectivity config             hub-and-spoke to hub-swc, use hub gateway
├── Security admin config           deny internet → 22, 3389, 445, 135; always-allow monitoring
└── Routing config                  0.0.0.0/0 → hub firewall for corp-spokes
     └─ deployed to: swedencentral, westeurope
```

A subscription vended for a new workload gets a VNet from the IPAM pool with the right tag. Within minutes it is peered, protected and routed. The workload team never files a networking ticket.

### Without AVNM

Perfectly workable for small estates: peerings, route tables and NSGs are created by the subscription vending or spoke IaC module. AVNM becomes worthwhile when the number of spokes, the rate of change, or the need for unoverridable rules grows.

### Cross-subscription and cross-tenant peering

The identity creating the peering needs *Network Contributor* (or the specific peering permissions) on **both** VNets. In a landing zone, the vending automation's identity holds that on the hub, so workload teams do not need rights in the connectivity subscription.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Hub-and-spoke or mesh | Hub-and-spoke with central inspection as the default. Add direct connectivity for specific spoke groups with high mutual traffic or latency sensitivity |
| Hub-and-spoke or Virtual WAN | Hub-and-spoke for control and custom needs; Virtual WAN for large global footprints and many branches (file 13) |
| Spoke-to-spoke through firewall or direct | Through the firewall across trust boundaries. Direct within a trusted group to save latency and firewall cost |
| Manual peering or AVNM | AVNM once spokes are numerous or created continuously |
| Security admin rules or policy-enforced NSGs | Admin rules for a small set of absolute organisational rules. NSGs for workload detail |
| Peering or Private Link between workloads | Private Link when one workload should expose a single service, not its network; also the answer for overlapping address spaces and cross-tenant sharing |

## Hands-on

```bash
RG=rg-peer-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Hub and two spokes
az network vnet create -g $RG -n vnet-hub     --address-prefixes 10.0.0.0/16  --subnet-name snet-shared --subnet-prefixes 10.0.1.0/24 -o none
az network vnet create -g $RG -n vnet-spoke-a --address-prefixes 10.10.0.0/16 --subnet-name snet-app    --subnet-prefixes 10.10.1.0/24 -o none
az network vnet create -g $RG -n vnet-spoke-b --address-prefixes 10.20.0.0/16 --subnet-name snet-app    --subnet-prefixes 10.20.1.0/24 -o none

# 2. Peer hub <-> spokes (two links each)
for S in a b; do
  az network vnet peering create -g $RG -n hub-to-spoke-$S --vnet-name vnet-hub --remote-vnet vnet-spoke-$S \
    --allow-vnet-access true --allow-forwarded-traffic true -o none
  az network vnet peering create -g $RG -n spoke-$S-to-hub --vnet-name vnet-spoke-$S --remote-vnet vnet-hub \
    --allow-vnet-access true --allow-forwarded-traffic true -o none
done

# 3. Check state
az network vnet peering list -g $RG --vnet-name vnet-hub \
  --query "[].{Name:name, State:peeringState, Sync:peeringSyncLevel, Remote:remoteAddressSpace.addressPrefixes[0]}" -o table

# 4. Try to peer an overlapping network and read the error
az network vnet create -g $RG -n vnet-overlap --address-prefixes 10.10.0.0/16 -o none
az network vnet peering create -g $RG -n hub-to-overlap --vnet-name vnet-hub --remote-vnet vnet-overlap --allow-vnet-access true

# 5. Change a spoke's address space and sync the peering
az network vnet update -g $RG -n vnet-spoke-a --address-prefixes 10.10.0.0/16 10.11.0.0/16 -o none
az network vnet peering sync -g $RG -n hub-to-spoke-a --vnet-name vnet-hub -o none

az group delete -n $RG --yes --no-wait
```

With VMs in each VNet, confirm that hub ↔ spoke works and spoke A ↔ spoke B does not, then inspect effective routes to see why.

AVNM (portal, optional): create a network manager scoped to your subscription, a network group with dynamic membership on a tag, a hub-and-spoke connectivity configuration and a security admin rule denying inbound 3389 from the internet. Deploy to one region, tag a VNet, and watch it join. Remove the deployment afterwards.

## Security, cost and reliability notes

- **Security:** peering extends the `VirtualNetwork` tag, so default NSG rules allow traffic between peered networks. Add explicit denies.
- **Security:** who can create peerings is a security question. A stray peering between a sandbox and a production spoke bypasses the hub. Restrict with RBAC and deny non-approved peerings with Azure Policy.
- **Security:** `Always Allow` admin rules bypass NSGs. Use them rarely and narrowly.
- **Cost:** peering traffic is charged per GB both ways. Chatty cross-spoke or cross-region designs add up, and traffic through a hub firewall pays peering twice plus firewall processing.
- **Reliability:** peering has no single point of failure. The hub's shared services do; make the firewall and gateways zone-redundant.
- **Reliability:** AVNM changes take effect only on deployment. Stage by region.

## Common mistakes

- Expecting transitive routing through a hub with no router or routes.
- Creating one side of the peering only.
- Overlapping address spaces.
- Forgetting *allow forwarded traffic* on spokes, so firewall-routed packets are dropped.
- Setting *use remote gateways* when the hub has no gateway yet, or giving the spoke its own gateway.
- Not syncing a peering after changing address space.
- Assuming peering shares DNS.
- A full mesh of manual peerings between dozens of spokes.
- Editing AVNM configurations and forgetting to deploy them.

## Check yourself

1. What does "peering is not transitive" mean, and how do you connect two spokes?
2. Which peering settings enable a spoke to use the hub's ExpressRoute gateway?
3. What must be true of address spaces for peering?
4. In AVNM, what is the difference between static and dynamic network group membership?
5. How are security admin rules evaluated relative to NSGs, and what do Allow, Always Allow and Deny do?
6. What is a connected group?
7. When would you use Private Link instead of peering between two workloads?

<details>
<summary>Answers</summary>

1. A network peered to the hub cannot reach another network peered to the same hub through the peering alone. Connect spokes with a routing device in the hub plus routes, with direct peering or an AVNM connected group, or with Virtual WAN.
2. *Allow gateway transit* on the hub side and *use remote gateways* on the spoke side.
3. They must not overlap.
4. Static membership lists VNets explicitly; dynamic membership is determined by an Azure Policy condition such as a tag, so new VNets join automatically.
5. Before NSGs. Allow passes traffic on to NSG evaluation; Always Allow permits it and skips NSGs; Deny drops it and NSGs cannot override.
6. An AVNM mesh construct that gives all member VNets direct connectivity without individual peerings.
7. When only one service should be exposed, when address spaces overlap, or when sharing across tenants without network-level trust.

</details>

## References

- [Virtual network peering](https://learn.microsoft.com/azure/virtual-network/virtual-network-peering-overview)
- [Create, change, or delete a virtual network peering](https://learn.microsoft.com/azure/virtual-network/virtual-network-manage-peering)
- [Hub-spoke network topology in Azure](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke)
- [What is Azure Virtual Network Manager?](https://learn.microsoft.com/azure/virtual-network-manager/overview)
- [Connectivity configuration in Azure Virtual Network Manager](https://learn.microsoft.com/azure/virtual-network-manager/concept-connectivity-configuration)
- [Security admin rules in Azure Virtual Network Manager](https://learn.microsoft.com/azure/virtual-network-manager/concept-security-admins)
- [User-defined route management in Azure Virtual Network Manager](https://learn.microsoft.com/azure/virtual-network-manager/concept-user-defined-route)
- [Spoke-to-spoke networking](https://learn.microsoft.com/azure/architecture/networking/guide/spoke-to-spoke-networking)
