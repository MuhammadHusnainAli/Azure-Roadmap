# Routing

## Why it matters

NSGs decide whether a packet is allowed. Routing decides where it goes. Hub-and-spoke designs, forced tunnelling through a firewall, hybrid connectivity and network virtual appliances all depend on steering traffic with routes. Routing mistakes produce the hardest networking faults to diagnose: connections that hang, work in one direction only, or break when someone adds a peering.

## Key concepts

### System routes

Azure creates a route table for every subnet automatically.

| Destination | Next hop | Meaning |
|---|---|---|
| VNet address space | Virtual network | Traffic stays within the VNet |
| `0.0.0.0/0` | Internet | Everything else goes to the internet (subject to an outbound path existing) |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10` | None | Unused private ranges are dropped until you define them |

More system routes appear when you add features:

| Feature | Route added |
|---|---|
| VNet peering | Peered VNet's address space → VNet peering |
| Virtual network gateway | On-premises prefixes (from BGP or the local network gateway) → Virtual network gateway |
| Service endpoint | The service's public prefixes → VirtualNetworkServiceEndpoint |

### User-defined routes

A **route table** is a resource holding custom routes that you associate with one or more subnets. Routes in it override or add to system routes.

| Next hop type | Sends traffic to |
|---|---|
| **Virtual appliance** | A private IP you specify: Azure Firewall, an NVA, or an internal load balancer in front of NVAs |
| **Virtual network gateway** | The VPN gateway (not valid for ExpressRoute, which uses BGP) |
| **Virtual network** | Normal intra-VNet routing |
| **Internet** | Directly out |
| **None** | Dropped (a black hole) |

Each route has an address prefix, which can be a CIDR or a **service tag**.

A route table has a setting **Propagate gateway routes** (BGP route propagation). Turning it **off** on a subnet stops on-premises routes learned by the gateway from appearing there, which is how you make sure a spoke reaches on-premises only through the firewall.

### Route selection

1. **Longest prefix match wins.** A `/24` beats a `/16` beats `0.0.0.0/0`.
2. For identical prefixes, by source: **user-defined route** beats **BGP route** beats **system route**.

That is the entire algorithm, and applying it carefully resolves most routing puzzles. One exception worth knowing: system routes for VNet peering and service endpoints are preferred even if a BGP route has the same prefix.

### Forced tunnelling

Sending all internet-bound traffic through an inspection point:

```text
Route table on spoke subnets:
  0.0.0.0/0  →  Virtual appliance  →  10.0.1.4   (Azure Firewall private IP in the hub)
  Propagate gateway routes: No
```

Now everything leaving the spoke for an unknown destination hits the firewall. The firewall needs its own path out.

Variants:

- **Force tunnel to on-premises:** `0.0.0.0/0` advertised over ExpressRoute or VPN, so internet traffic exits through the corporate proxy. Adds latency and can break Azure platform dependencies unless exceptions are made.
- **Azure Firewall forced tunnelling:** the firewall itself sends internet traffic onward to another device, using a separate management subnet for its own control traffic.

### Asymmetric routing

Stateful devices (firewalls, NVAs) must see **both directions** of a flow. If the request goes through the firewall and the reply bypasses it, the firewall drops the connection or the client sees a reset.

Typical causes:

- A UDR on the source subnet but not on the destination subnet.
- Two firewalls in active-active without flow symmetry.
- Traffic from on-premises arriving through the gateway directly to the spoke while the return path goes through the firewall, because the `GatewaySubnet` has no route sending spoke-bound traffic to the firewall.
- Public IP on a VM whose default route points at a firewall: inbound arrives directly, outbound leaves through the firewall.

Rule: for any pair of networks that should be inspected, put routes on **both** sides, including the gateway subnet.

### Network virtual appliances

Third-party firewalls, SD-WAN and routers run as VMs. Requirements:

- **IP forwarding** enabled on the NVA's NIC, and in the guest OS.
- **High availability:** two or more NVAs behind an **internal Standard Load Balancer with HA ports**, which forwards all protocols and ports and keeps flows symmetric; or **Gateway Load Balancer** for transparent insertion in front of public endpoints; or active-passive with route updates.
- Routes pointing at the load balancer's frontend IP, not at a single NVA.

### Azure Route Server

A managed service that speaks **BGP** with your NVAs and programs the routes it learns into the VNet (and peered VNets), and tells NVAs about VNet prefixes. It removes the need to maintain UDRs by hand whenever an NVA's routes change.

| Aspect | Detail |
|---|---|
| Subnet | `RouteServerSubnet`, `/27` |
| Role | Control plane only. It exchanges routes; data does not flow through it |
| Use cases | SD-WAN NVAs advertising branch routes; dual-homed designs; route exchange between ExpressRoute and VPN gateways (branch-to-branch) |
| Availability | Zone-redundant, two instances |

### Routing in peered and hub-and-spoke networks

- **Peering is not transitive.** Spoke A peered to the hub and spoke B peered to the hub cannot reach each other unless something in the hub routes between them and both spokes have a route to it.
- **Spoke-to-spoke through a firewall:** each spoke has `0.0.0.0/0` (or the other spokes' summary prefix) → firewall. The firewall allows and forwards.
- **Gateway transit:** the hub's gateway serves spokes when peering is configured with *allow gateway transit* on the hub side and *use remote gateways* on the spoke side.
- **Gateway subnet route table:** add routes for spoke prefixes → firewall so that on-premises-to-spoke traffic is inspected and symmetric. Do not add `0.0.0.0/0` to the `GatewaySubnet`.

### Service tags in routes

A UDR's prefix can be a service tag, for example `AzureMonitor` → Internet, to let specific platform traffic bypass the firewall while everything else is forced through it. Use sparingly and document why.

### Limits and management

- 400 routes per route table by default, with higher limits available when using Azure Virtual Network Manager's UDR management.
- **AVNM routing configurations** generate and maintain route tables across many spokes from a single definition (file 04).

## How it works on Azure

### Reading effective routes

The truth for any NIC is its **effective routes**: the merged result of system, BGP and user-defined routes.

```bash
az network nic show-effective-route-table -g <rg> -n <nic-name> -o table
```

| Column | Tells you |
|---|---|
| Source | Default (system), User, VirtualNetworkGateway (BGP) |
| State | Active or Invalid (overridden) |
| Address prefix | The destination |
| Next hop type and IP | Where it goes |

Network Watcher's **Next hop** answers "where would a packet from this VM to that IP go?" directly (file 17).

### A hub-and-spoke routing plan

| Subnet | Route table contents | Gateway propagation |
|---|---|---|
| Spoke workload subnets | `0.0.0.0/0` → firewall | Off |
| Spoke private endpoint subnet (policies enabled) | Usually none needed; add routes if PE traffic must be inspected | Off |
| `GatewaySubnet` | Each spoke prefix (or a summary) → firewall | On (required) |
| `AzureFirewallSubnet` | `0.0.0.0/0` → Internet (default); no UDR sending traffic back to itself | On |
| `AzureBastionSubnet` | No UDR with a default route (not supported) | – |
| Application Gateway subnet | Restrictions on `0.0.0.0/0` UDRs apply; follow the service's guidance, often routing only private ranges to the firewall | – |

Several platform subnets restrict or forbid UDRs. Check before applying a blanket route table.

### Private endpoints and routing

A private endpoint creates a `/32` route for its IP in the VNet and peered VNets. Because `/32` is the longest possible match, traffic to a private endpoint **bypasses** a `0.0.0.0/0` or summary UDR and goes straight to it. To inspect private endpoint traffic through a firewall, enable private endpoint network policies for route tables on the PE subnet and add a UDR for the PE subnet's prefix; the platform then honours it. Decide whether that inspection is worth the complexity.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Central inspection of egress | Standard for landing zones: `0.0.0.0/0` → firewall on spokes |
| Inspect spoke-to-spoke | Yes for different workloads or trust levels. It adds latency and firewall cost; some designs peer trusted spokes directly |
| Inspect private endpoint traffic | Often unnecessary within a workload; consider for cross-workload access |
| Static UDRs or Route Server | UDRs for Azure Firewall. Route Server when NVAs or SD-WAN advertise changing routes |
| Hand-managed route tables or AVNM / Virtual WAN | Automate once you have more than a few spokes. Virtual WAN routing intent removes UDR management entirely |
| Force tunnel to on-premises | Only when policy demands. Prefer inspecting in Azure |

## Hands-on

Uses the VNet from file 01.

```bash
RG=rg-vnet-lab; VNET=vnet-lab-swc-001

# 1. A route table that forces egress to a (hypothetical) firewall at 10.0.1.4
az network route-table create -g $RG -n rt-spoke-egress --disable-bgp-route-propagation true -o none
az network route-table route create -g $RG --route-table-name rt-spoke-egress -n default-to-firewall \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4 -o none

# 2. A service-tag route that bypasses the firewall, and a black hole
az network route-table route create -g $RG --route-table-name rt-spoke-egress -n monitor-direct \
  --address-prefix AzureMonitor --next-hop-type Internet -o none
az network route-table route create -g $RG --route-table-name rt-spoke-egress -n drop-legacy \
  --address-prefix 192.168.50.0/24 --next-hop-type None -o none

# 3. Associate and review
az network vnet subnet update -g $RG --vnet-name $VNET -n snet-mgmt --route-table rt-spoke-egress -o none
az network route-table route list -g $RG --route-table-name rt-spoke-egress \
  --query "[].{Name:name, Prefix:addressPrefix, NextHop:nextHopType, IP:nextHopIpAddress}" -o table
```

With a VM in `snet-mgmt`, run `az network nic show-effective-route-table` and identify which routes are *User*, which system routes became *Invalid*, and what happens to a packet for `8.8.8.8`, `10.20.3.5` and `192.168.50.10`.

Paper exercises:

1. A spoke subnet has routes `0.0.0.0/0 → firewall` (UDR), `10.0.0.0/8 → gateway` (BGP), `10.20.0.0/22 → VNet` (system) and a private endpoint at `10.20.3.5/32`. State the next hop for `10.20.3.5`, `10.99.1.1` and `52.1.2.3`.
2. On-premises users reach a spoke VM, but replies never arrive. Spokes route `0.0.0.0/0` to the firewall. What is missing?

<details>
<summary>Answers</summary>

1. `10.20.3.5`: the private endpoint directly (`/32` is the longest match). `10.99.1.1`: the gateway (`/8` beats `/0`). `52.1.2.3`: the firewall.
2. A route on the `GatewaySubnet` sending the spoke prefix to the firewall. Without it, inbound traffic goes gateway → spoke directly and the reply goes spoke → firewall, which drops it as asymmetric.

</details>

## Security, cost and reliability notes

- **Security:** a route table is a security control when it enforces inspection. Protect it with RBAC, policy or a deployment stack so a workload team cannot detach it.
- **Security:** disabling gateway route propagation on spokes prevents a path to on-premises that bypasses the firewall.
- **Cost:** traffic through a firewall incurs data processing charges; inspect what needs inspecting.
- **Reliability:** a UDR pointing at an NVA that is down black-holes traffic. Use a load balancer with health probes in front of NVAs.
- **Reliability:** a wrong `0.0.0.0/0` route can cut a subnet off from platform services (management, DNS, monitoring). Test on one subnet first.

## Common mistakes

- Expecting spoke-to-spoke traffic to work across a hub with no routing device.
- Routes on one side only, causing asymmetric flows.
- No route on the `GatewaySubnet` for spoke prefixes.
- A `0.0.0.0/0` UDR on `GatewaySubnet`, `AzureBastionSubnet`, or a platform subnet that forbids it.
- Pointing a route at a single NVA instance.
- Forgetting IP forwarding on NVA NICs.
- Assuming a default route UDR captures private endpoint traffic.
- Leaving gateway route propagation on in spokes that should be forced through a firewall.
- Diagnosing by guesswork instead of reading effective routes.

## Check yourself

1. What are the two rules of route selection?
2. Name the five next hop types for a user-defined route.
3. What does disabling gateway route propagation do?
4. What causes asymmetric routing, and why does it break connections?
5. Why does a `0.0.0.0/0` UDR not capture traffic to a private endpoint?
6. What does Azure Route Server do, and does data pass through it?
7. How do you make NVAs highly available?

<details>
<summary>Answers</summary>

1. Longest prefix match first; for equal prefixes, user-defined beats BGP beats system.
2. Virtual appliance, virtual network gateway, virtual network, internet, none.
3. Stops routes learned by the virtual network gateway (from on-premises) being added to the subnet's effective routes.
4. Request and reply take different paths so a stateful device sees only one direction and drops the flow.
5. The private endpoint injects a `/32` route, which is more specific than the default route.
6. It exchanges routes over BGP between NVAs and the virtual network and programs learned routes automatically. It is control plane only; data does not flow through it.
7. Run two or more behind an internal Standard Load Balancer with HA ports (or a Gateway Load Balancer) and route to the load balancer's frontend IP.

</details>

## References

- [Virtual network traffic routing](https://learn.microsoft.com/azure/virtual-network/virtual-networks-udr-overview)
- [Create, change, or delete a route table](https://learn.microsoft.com/azure/virtual-network/manage-route-table)
- [What is Azure Route Server?](https://learn.microsoft.com/azure/route-server/overview)
- [Deploy highly available NVAs](https://learn.microsoft.com/azure/architecture/networking/guide/network-virtual-appliance-high-availability)
- [Hub-spoke network topology in Azure](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke)
- [Azure Firewall forced tunneling](https://learn.microsoft.com/azure/firewall/forced-tunneling)
- [Use Azure Firewall to inspect traffic destined to a private endpoint](https://learn.microsoft.com/azure/private-link/inspect-traffic-with-azure-firewall)
- [Diagnose a virtual machine routing problem](https://learn.microsoft.com/azure/virtual-network/diagnose-network-routing-problem)
