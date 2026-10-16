# Virtual WAN

> Topic: 05 · Networking · Tracks: E – A ● C ○ · Status checked: 2026-10

## Why it matters

Hub-and-spoke gives you full control and makes you build everything: the hub network, gateways, route tables on every spoke, peering between regional hubs. Azure Virtual WAN is the managed alternative. Microsoft runs the hub and its routing, and you attach networks to it. It is one of the two network topologies in the Azure landing zone reference, and the usual choice for organisations with many branches or regions.

## Key concepts

### What it is

A networking service that brings site-to-site VPN, point-to-site VPN, ExpressRoute, VNet connectivity, routing and security together around **Microsoft-managed virtual hubs**.

| Resource | Meaning |
|---|---|
| **Virtual WAN** | The global container |
| **Virtual hub** | A Microsoft-managed VNet in a region with a managed router. You choose its address space (a `/23` is recommended) but cannot deploy your own resources into it |
| **Hub gateways** | Site-to-site VPN, point-to-site VPN and ExpressRoute gateways deployed in the hub, sized in **scale units** |
| **Virtual network connection** | Attaches a spoke VNet to the hub (a managed peering) |
| **Hub route tables** | Routing configuration, with association and propagation settings per connection |
| **Secured virtual hub** | A hub with Azure Firewall or a partner security solution, managed through Firewall Manager |

### Types

| | **Basic** | **Standard** |
|---|---|---|
| Site-to-site VPN | Yes | Yes |
| ExpressRoute, point-to-site VPN | No | Yes |
| VNet-to-VNet transit through the hub | No | Yes |
| Hub-to-hub (inter-region) transit | No | Yes |
| Azure Firewall and NVAs in the hub | No | Yes |

Standard is the real product. Basic can be upgraded to Standard, not downgraded.

### What "managed" gives you

- **Any-to-any transit by default.** Connected VNets, branches, ExpressRoute circuits and remote users can reach each other through the hub without you writing routes.
- **Automatic hub-to-hub mesh.** Hubs in different regions are connected over the Microsoft backbone; a VNet in one region reaches a branch connected to another.
- **No UDRs on spokes** for hub-routed traffic; the hub router programs routes.
- **Scale units** instead of gateway SKUs, adjustable without redeployment.
- **Partner integration.** Many SD-WAN and VPN devices can configure their connection to a hub automatically.

### Routing

Each connection (VNet, VPN site, ExpressRoute circuit, P2S configuration) has two settings:

| Setting | Meaning |
|---|---|
| **Association** | The one route table this connection **uses** to decide where to send traffic |
| **Propagation** | The route tables (and **labels**, which group tables across hubs) this connection **advertises its prefixes to** |

By default everything associates with and propagates to the **default route table**, giving full mesh. Custom route tables create **isolation**: for example, spokes that can reach shared services and on-premises but not each other.

**Routing intent and routing policies** are the simpler modern way to steer traffic through a security solution. You declare:

- **Internet traffic policy:** send internet-bound traffic to the firewall.
- **Private traffic policy:** send all private traffic (VNet-to-VNet, branch-to-VNet, inter-hub) to the firewall.

The hub then programs everything, including inter-hub inspection, with no custom route tables or spoke UDRs. Use routing intent for new secured hubs unless you need something it does not support.

Other routing features: **static routes** on connections (for an NVA in a spoke), **BGP peering** with NVAs in spokes, **hub routing preference** (prefer ExpressRoute, VPN or AS path when routes conflict), and **route maps** to filter and modify routes.

### Security in the hub

| Option | Notes |
|---|---|
| **Azure Firewall** | Deployed into the hub through Firewall Manager; uses firewall policies as usual |
| **Integrated NVAs** | Selected partner firewalls and SD-WAN appliances deployed as managed resources in the hub |
| **SaaS security solutions** | For example a vendor's cloud next-generation firewall as a service in the hub |
| **Third-party security-as-a-service** | For internet-bound traffic |

### Constraints

Because Microsoft owns the hub, some things you would do in a self-managed hub are not possible:

| You cannot put in a virtual hub | Where it goes instead |
|---|---|
| Azure Bastion | A spoke VNet |
| DNS Private Resolver, domain controllers, jump boxes | A shared services spoke |
| Application Gateway, API Management | Workload or shared spokes |
| Arbitrary VMs or NVAs (beyond supported integrated ones) | A spoke, with static routes or BGP peering to the hub |
| Your own private DNS zone links to the hub VNet | Link zones to the spoke hosting the DNS resolver or firewall, and point other VNets at it |

Other points:

- Routing customisation is bounded by what hub route tables and routing intent expose.
- There is one firewall instance per hub; some advanced multi-NVA designs are harder.
- A VNet connects to **one** hub in a given Virtual WAN.
- The hub router has throughput capacity defined by **routing infrastructure units**, which you can scale.

## How it works on Azure

### Hub-and-spoke compared with Virtual WAN

| | **Hub-and-spoke (customer-managed)** | **Virtual WAN (Microsoft-managed)** |
|---|---|---|
| Hub network | A VNet you own; deploy anything in it | Managed; limited to supported services |
| Spoke-to-spoke transit | Needs a firewall or NVA and UDRs | Built in |
| Routing | UDRs you write (or AVNM) | Hub router; routing intent |
| Cross-region | Global peering between hubs, routes managed by you | Automatic hub-to-hub |
| Branch scale | Limited by gateway SKUs and manual configuration | Designed for hundreds to thousands of sites |
| Shared services placement | In the hub | In a shared services spoke |
| Flexibility | Maximum | Constrained |
| Operational effort | Higher | Lower |
| Cost model | Pay for each component you deploy | Hub base fee, scale units, connection units, data processing |

**Choose Virtual WAN when** you have many branches or remote users, several regions needing transit, SD-WAN integration, or limited appetite for managing routing.

**Choose hub-and-spoke when** you need resources or NVAs in the hub that Virtual WAN does not support, fine-grained routing control, or you have a small footprint in one or two regions where the managed hub's cost is not justified.

Both are supported by the landing zone accelerators. Migration from one to the other is possible and is a project.

### A Virtual WAN landing zone

```text
Virtual WAN (Standard)
├── Hub: Sweden Central   [Azure Firewall · ER gateway · S2S VPN · P2S VPN]   routing intent: private + internet → firewall
│     ├── Shared services spoke   (DNS Private Resolver, Bastion, domain controllers)
│     ├── Spoke: hr-assistant-prod
│     └── Spoke: dataplatform-prod
├── Hub: West Europe      [Azure Firewall · ER gateway]
│     └── Spokes …
├── ExpressRoute circuits (two peering locations) ── connected to both hubs
└── Branch sites (SD-WAN / VPN)
```

DNS design in this model: private DNS zones are linked to the **shared services spoke**, where the DNS Private Resolver lives. All other VNets set their DNS servers to the resolver's inbound endpoint (or the hub firewall's DNS proxy, which forwards to it).

### AI workloads on Virtual WAN

Nothing changes for the workload itself. A network-isolated Foundry spoke connects to the hub like any other, with:

- routing intent sending its egress to the hub firewall, where the FQDN allow-list applies (file 10);
- private endpoints in the spoke, resolved through the shared services DNS;
- on-premises reachability through the hub's ExpressRoute or VPN gateway.

Private endpoint traffic within the same spoke stays local. With private traffic routing policies enabled, private endpoint traffic between spokes is inspected by the hub firewall, provided private endpoint network policies are enabled on the endpoint subnets.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Virtual WAN or hub-and-spoke | See the comparison. Decide early; it shapes everything else |
| Routing intent or custom route tables | Routing intent for secured hubs. Custom route tables only for isolation patterns it cannot express |
| One hub per region | Yes, in each region with workloads or significant user populations |
| Where shared services go | A dedicated shared services spoke per region |
| Firewall choice | Azure Firewall for native simplicity; integrated NVA or SaaS firewall for a vendor standard |
| Hub address space | `/23` or larger, non-overlapping, never to be reused |

**Cost.** A hub has an hourly base charge even when empty, plus scale units for each gateway, connection units, firewall charges and per-GB data processing. For a small single-region environment this is more than a self-built hub. For a multi-region, multi-branch network it is frequently less than the equivalent set of gateways, plus much less engineering time.

## Hands-on

A hub with gateways is slow to deploy and bills hourly. This lab creates a hub without gateways (about 10 to 30 minutes to provision) to explore routing. Delete it promptly.

```bash
RG=rg-vwan-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none
az extension add -n virtual-wan 2>/dev/null

az network vwan create -g $RG -n vwan-lab --type Standard -o none
az network vhub create -g $RG -n vhub-swc --vwan vwan-lab --address-prefix 10.100.0.0/23 -l $LOC --no-wait

for S in a b; do
  P=$([ $S = a ] && echo 10.110 || echo 10.120)
  az network vnet create -g $RG -n vnet-spoke-$S --address-prefixes $P.0.0/16 --subnet-name snet-app --subnet-prefixes $P.1.0/24 -o none
done

# when the hub's routingState is "Provisioned":
az network vhub show -g $RG -n vhub-swc --query "{state:provisioningState, routing:routingState}"
for S in a b; do
  az network vhub connection create -g $RG --vhub-name vhub-swc -n conn-spoke-$S \
    --remote-vnet vnet-spoke-$S -o none
done

# Inspect routing: default route table, and what each connection associates with and propagates to
az network vhub route-table list -g $RG --vhub-name vhub-swc --query "[].{Name:name, Labels:labels}" -o table
az network vhub connection show -g $RG --vhub-name vhub-swc -n conn-spoke-a \
  --query "routingConfiguration.{assoc:associatedRouteTable.id, prop:propagatedRouteTables.ids[].id}"
az network vhub get-effective-routes -g $RG -n vhub-swc \
  --resource-type RouteTable \
  --resource-id $(az network vhub route-table show -g $RG --vhub-name vhub-swc -n defaultRouteTable --query id -o tsv)

az group delete -n $RG --yes --no-wait
```

Note that the two spokes can reach each other with no UDRs or direct peering. In the portal, look at *Routing → Routing Intent and Routing Policies* and *Effective routes*.

Design exercise: an organisation has 40 branch offices on SD-WAN, two data centres with ExpressRoute, and workloads in three Azure regions. Sketch the Virtual WAN design: hubs, gateways, where the firewall, DNS resolver and Bastion go, and the routing policy.

## Security, cost and reliability notes

- **Security:** default any-to-any connectivity is convenient and permissive. Add a secured hub with routing intent, or isolation route tables, before connecting production workloads.
- **Security:** you cannot attach an NSG to the hub. Controls are the hub firewall and NSGs in spokes.
- **Cost:** hubs and gateways bill continuously. Delete lab hubs.
- **Reliability:** hubs and their gateways are zone-redundant where the region supports it. For regional resilience deploy hubs in multiple regions; they mesh automatically.
- **Reliability:** scale units and routing infrastructure units cap throughput. Monitor and scale ahead of demand.

## Common mistakes

- Trying to deploy Bastion, a DNS resolver or domain controllers into the virtual hub.
- Choosing Basic and discovering it lacks ExpressRoute and transit.
- Building custom route tables for something routing intent does simply.
- Linking private DNS zones only to workload spokes and wondering why central resolution fails.
- An undersized or overlapping hub address range.
- Assuming connected means inspected; without a secured hub, traffic is not filtered.
- Forgetting a test hub is running.
- Selecting Virtual WAN for a tiny single-region environment where a simple hub would be cheaper and more flexible.

## Check yourself

1. What does Microsoft manage in a virtual hub that you manage in a traditional hub?
2. Name three capabilities that require the Standard type.
3. What do *association* and *propagation* mean for a connection?
4. What does routing intent do?
5. Where do Bastion and the DNS Private Resolver go in a Virtual WAN design?
6. Give two reasons to choose Virtual WAN and two to choose hub-and-spoke.

<details>
<summary>Answers</summary>

1. The hub virtual network, its router and routing between connections, including inter-hub connectivity.
2. Any three of: ExpressRoute, point-to-site VPN, VNet-to-VNet transit, hub-to-hub transit, Azure Firewall or NVAs in the hub.
3. Association is the route table a connection uses for its outbound routing decisions. Propagation is where the connection advertises its own prefixes.
4. Declares that internet-bound and/or private traffic must go through the hub's security solution, and programs the routing for it automatically, including across hubs.
5. In a shared services spoke VNet connected to the hub.
6. Virtual WAN: many branches or regions, built-in transit and less routing management. Hub-and-spoke: need for unsupported resources or NVAs in the hub, maximum routing control, or a small footprint where cost matters.

</details>

## References

- [What is Azure Virtual WAN?](https://learn.microsoft.com/azure/virtual-wan/virtual-wan-about)
- [About virtual hub routing](https://learn.microsoft.com/azure/virtual-wan/about-virtual-hub-routing)
- [How to configure Virtual WAN hub routing intent and routing policies](https://learn.microsoft.com/azure/virtual-wan/how-to-routing-policies)
- [Virtual WAN network topology (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/virtual-wan-network-topology)
- [Migrate to Azure Virtual WAN](https://learn.microsoft.com/azure/virtual-wan/migrate-from-hub-spoke-topology)
- [Hub-spoke network topology with Azure Virtual WAN](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke-virtual-wan-architecture)
- [Private Link and DNS in Azure Virtual WAN](https://learn.microsoft.com/azure/architecture/networking/guide/private-link-virtual-wan-dns-guide)
- [Virtual WAN FAQ](https://learn.microsoft.com/azure/virtual-wan/virtual-wan-faq)
