# VPN Gateway and ExpressRoute

## Why it matters

Most organisations' data, users and systems are not all in Azure. Hybrid connectivity links the corporate network to Azure so that an AI application can reach an on-premises database, users can reach an internal assistant over the private network, and private endpoints resolve and route from the office. The choice between VPN and ExpressRoute affects performance, resilience and cost for years, and circuits have lead times measured in weeks.

## Key concepts

### The two options

| | **VPN Gateway** | **ExpressRoute** |
|---|---|---|
| Path | Encrypted IPsec tunnel over the **public internet** | **Private** connection through a connectivity provider. Does not traverse the internet |
| Bandwidth | Up to roughly 10 Gbps aggregate on the largest SKUs; per-tunnel limits lower | 50 Mbps to 10 Gbps through providers; 10 and 100 Gbps with ExpressRoute Direct |
| Latency and consistency | Variable, internet-dependent | Predictable, low |
| Encryption | Built in (IPsec/IKE) | **Not encrypted by default**. Add MACsec (Direct) or IPsec over ExpressRoute if required |
| SLA | On the gateway | On the circuit and gateway, with higher guarantees for resilient designs |
| Setup time | Minutes to hours | Weeks, with a provider |
| Cost | Gateway per hour plus egress | Circuit monthly fee (by bandwidth and SKU), provider charges, gateway per hour; metered or unlimited data |
| Typical use | Small sites, dev/test, backup path, quick start | Production hybrid, large data transfer, latency-sensitive and regulated workloads |

Both terminate on a **virtual network gateway** in a subnet named `GatewaySubnet` (`/27` or larger). A VNet can host one VPN gateway and one ExpressRoute gateway side by side.

### VPN Gateway

**Connection types**

| Type | Connects |
|---|---|
| **Site-to-site (S2S)** | An on-premises VPN device to Azure, IPsec/IKEv2 |
| **Point-to-site (P2S)** | An individual client device to Azure. Protocols: OpenVPN, IKEv2, SSTP. Authentication: **Microsoft Entra ID** (with Conditional Access and MFA), certificates, or RADIUS |
| **VNet-to-VNet** | Two Azure VNets through gateways (peering is usually better) |

**Building blocks for site-to-site**

| Resource | Represents |
|---|---|
| Virtual network gateway | The Azure end |
| Local network gateway | Your on-premises device: its public IP and the address prefixes behind it (or BGP settings) |
| Connection | The tunnel between them, with a shared key and optional custom IPsec/IKE policy |

**SKUs.** The generation 2 availability-zone SKUs, `VpnGw2AZ` to `VpnGw5AZ`, are the standard choice; they differ in throughput, tunnel count and P2S connections. Non-AZ SKUs are being phased out in favour of AZ SKUs. The *Basic* SKU is for development only and lacks BGP, zone redundancy and modern features. Use **route-based** VPN; policy-based is legacy.

**High availability**

| Mode | Behaviour |
|---|---|
| **Active-standby** (default) | Two instances; failover takes seconds for planned events and up to a minute or more for unplanned |
| **Active-active** | Both instances have public IPs and tunnels. With two on-premises devices you get four tunnels and the best resilience |
| **Zone-redundant** (AZ SKUs) | Instances spread across availability zones |

**BGP** exchanges routes dynamically with on-premises, removes the need to maintain static prefixes, and enables automatic failover between tunnels and transit routing. Use it for anything beyond a single simple tunnel.

### ExpressRoute

**Concepts**

| Term | Meaning |
|---|---|
| **Circuit** | The logical connection between your network and Microsoft through a provider, identified by a service key |
| **Peering location** | The facility (an exchange or colocation site) where the provider meets Microsoft. Not the same as an Azure region |
| **Connectivity provider** | The carrier or exchange delivering the circuit |
| **Private peering** | Reaches your virtual networks' private address space |
| **Microsoft peering** | Reaches Microsoft public services (Microsoft 365 with authorisation, Azure PaaS public endpoints) over the circuit, using public IPs and route filters |
| **ExpressRoute gateway** | The virtual network gateway that connects a VNet to a circuit |

Each circuit has **two physical connections** to two Microsoft edge routers for redundancy. Run BGP on both; do not treat one as spare capacity you can ignore.

**Connectivity models**

| Model | Description |
|---|---|
| Cloud exchange colocation | Cross-connect at a facility where you already have presence |
| Point-to-point Ethernet | A dedicated link from your site |
| Any-to-any (IP VPN) | Azure joins your existing MPLS WAN as another site |
| **ExpressRoute Direct** | You connect directly to Microsoft's routers on dedicated 10 or 100 Gbps port pairs, then create circuits on them. Supports MACsec |

**Circuit SKUs**

| SKU | Reach |
|---|---|
| **Local** | Azure regions at or near the peering location only. Unlimited data included, no egress charge |
| **Standard** | All regions in the same geopolitical area |
| **Premium** | All regions globally, more route prefixes, more VNet links |

Billing is **metered** (pay for outbound data) or **unlimited** (flat fee).

**Gateway SKUs.** Zone-redundant `ErGw1AZ` to `ErGw3AZ`, and the autoscaling **scalable gateway** (`ErGwScale`). Higher SKUs give more throughput and enable **FastPath**.

**Features**

| Feature | Purpose |
|---|---|
| **FastPath** | Sends data-plane traffic directly to VMs, bypassing the gateway, for higher throughput and lower latency. Support for UDRs and private endpoints over FastPath has conditions; check current limitations |
| **Global Reach** | Links two ExpressRoute circuits so your sites communicate through Microsoft's backbone |
| **BFD** | Fast link-failure detection for quicker BGP convergence |
| **MACsec** | Layer 2 encryption between your edge and Microsoft's on ExpressRoute Direct |
| **Traffic Collector / Insights** | Flow sampling and topology views |
| **Route filters** | Select which Microsoft service prefixes are advertised on Microsoft peering |

### Resilience tiers for ExpressRoute

Microsoft's guidance defines three levels:

| Level | Design | Protects against |
|---|---|---|
| **Standard** | One circuit at one peering location (two links to two routers) | A link or device failure. **Not** a peering location outage |
| **High** | **ExpressRoute Metro**: one circuit whose two links land at **two different peering locations within the same metropolitan area** | A single site failure within the metro |
| **Maximum** | **Two circuits at two different peering locations**, ideally through different providers, both connected to the gateway | A complete peering location outage |

For production, design for **maximum** resiliency. A single circuit at a single location has been the root cause of many serious hybrid outages. Azure provides a guided deployment experience and a resiliency validation tool for this.

**Backup with VPN.** A site-to-site VPN can coexist as a lower-cost failover path. ExpressRoute routes are preferred over VPN routes when both advertise the same prefixes. Remember the bandwidth and latency difference when the backup is in use.

### Encryption over ExpressRoute

ExpressRoute is private but not encrypted. Options when policy demands encryption in transit across the provider network:

| Option | Layer | Notes |
|---|---|---|
| **TLS at the application** | 7 | Often sufficient and simplest. Azure services use TLS anyway |
| **MACsec** | 2 | ExpressRoute Direct only; encrypts the physical links |
| **IPsec VPN over ExpressRoute private peering** | 3 | A VPN gateway tunnel across the circuit. Throughput limited by the VPN gateway |

### Remote access for individuals

| Option | Notes |
|---|---|
| **Point-to-site VPN** with Entra authentication | Network-level access for administrators and developers; apply Conditional Access |
| **Azure Bastion** | Browser or native-client access to VMs without a VPN (file 14) |
| **Microsoft Entra Private Access** | Per-application Zero Trust access, replacing VPN for users (topic 02) |

## How it works on Azure

### Placement in a landing zone

```text
On-premises ══ ExpressRoute (two circuits, two peering locations) ══╗
On-premises ── S2S VPN (backup) ────────────────────────────────────╣
                                                                    ▼
                                    Hub VNet: GatewaySubnet ─ ExpressRoute gateway + VPN gateway
                                              AzureFirewallSubnet ─ firewall
                                              DNS Private Resolver
                                                    │ peering (use remote gateways)
                                              Spoke VNets (workloads, private endpoints)
```

- Gateways live in the **connectivity subscription's hub**. Spokes use them through peering with gateway transit.
- A route table on `GatewaySubnet` sends spoke prefixes to the firewall, so hybrid traffic is inspected and symmetric (file 03).
- **DNS** must resolve Azure private names from on-premises through the resolver's inbound endpoint (file 06). Connectivity without name resolution is half a solution.
- A circuit can link to several VNets, and a VNet to several circuits.

### What hybrid connectivity enables for AI workloads

| Need | How |
|---|---|
| Users on the corporate network reach an internal assistant | Private-frontend Application Gateway or internal ingress, reached over the circuit |
| The application reaches on-premises data (SQL Server, file shares, APIs) | Routes through the gateway; firewall rules; on-premises firewall rules for Azure ranges |
| Indexing on-premises content into Azure AI Search | A pipeline running in Azure with line of sight (Data Factory with a self-hosted integration runtime, or a job in the VNet) pulling data across the circuit |
| On-premises applications call a private Foundry endpoint | Private endpoint in Azure, reachable over private peering, with DNS forwarding |
| Large one-off data transfer | ExpressRoute bandwidth, or Data Box for very large volumes |
| Developers reach private resources | P2S VPN, Bastion, a cloud PC in the VNet (Windows 365 or Azure Virtual Desktop), or Private Access |

Latency matters for retrieval: an agent that makes several calls per turn to an on-premises system across a VPN can feel slow. Replicating data to Azure, or caching, is often the better design.

### Sizing

Estimate: sustained bandwidth, peak bandwidth, largest data transfers (initial loads, backups, index builds), latency tolerance, and growth. Circuit bandwidth can usually be increased without downtime; decreasing may require recreating the circuit.

### Monitoring

| Tool | Shows |
|---|---|
| Gateway metrics | Tunnel bandwidth, packet drops, BGP peer status, P2S connection count |
| ExpressRoute metrics | Circuit bits in and out, BGP availability, ARP availability |
| **Connection Monitor** | End-to-end reachability and latency between on-premises and Azure |
| **ExpressRoute Insights / Network Insights** | Topology and health |
| VPN diagnostics and troubleshoot | Tunnel negotiation logs |
| Resource health and Service Health | Platform issues and planned maintenance |

Alert on BGP session drops and tunnel disconnects. Test failover on a schedule.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| VPN or ExpressRoute | VPN to start, for small sites and as backup. ExpressRoute for production hybrid with meaningful bandwidth, latency or compliance needs |
| ExpressRoute resiliency level | Maximum for production. Accept standard only with a documented risk decision and a VPN backup |
| Local, Standard or Premium | Local when your region is at the peering location and data volumes are high. Premium for global reach or many VNet links |
| Metered or unlimited | Unlimited when outbound volume is high and steady; do the arithmetic |
| ExpressRoute provider or Direct | Direct for very high bandwidth, many circuits, or MACsec |
| Hub-and-spoke gateways or Virtual WAN | Virtual WAN when there are many sites and regions (file 13) |
| Encrypt over ExpressRoute | Application-layer TLS usually suffices; add MACsec or IPsec for a specific requirement |
| How much to keep on-premises | Every on-premises dependency of a cloud workload is latency and a failure domain. Move or replicate data where practical |

## Hands-on

A VPN gateway takes 30 to 45 minutes to deploy and bills hourly; ExpressRoute needs a provider. Do the review path unless you have time and budget.

**Review path**

1. In the portal, open the create wizard for a **virtual network gateway**. Compare VPN and ExpressRoute gateway SKUs, note zone options, and cancel.
2. Open the create wizard for an **ExpressRoute circuit**. Select the resiliency option and read how *maximum*, *high* and *standard* differ. Look at the peering locations and providers for your nearest city. Cancel.
3. Read the ExpressRoute locations and providers page and find which peering locations serve your region and whether *Local* SKU is available there.
4. Design exercise: a company with a head office and one data centre wants private access to an AI application in Sweden Central and to index a 2 TB on-premises document store. Specify: connectivity type, resiliency design, gateway SKUs, circuit SKU and billing model, backup path, DNS design, and how the initial 2 TB gets to Azure.

**Deploy path (optional)**: simulate site-to-site with two VNets.

```bash
RG=rg-vpn-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none
for N in azure onprem; do
  P=$([ $N = azure ] && echo 10.50 || echo 192.168)
  az network vnet create -g $RG -n vnet-$N --address-prefixes $P.0.0/16 --subnet-name GatewaySubnet --subnet-prefixes $P.255.0/27 -o none
  az network public-ip create -g $RG -n pip-vgw-$N --sku Standard --zone 1 2 3 -o none
  az network vnet-gateway create -g $RG -n vgw-$N --vnet vnet-$N --public-ip-addresses pip-vgw-$N \
    --gateway-type Vpn --vpn-type RouteBased --sku VpnGw2AZ --vpn-gateway-generation Generation2 --no-wait
done
# wait ~40 minutes:  az network vnet-gateway list -g $RG --query "[].{n:name, s:provisioningState}" -o table

az network vpn-connection create -g $RG -n azure-to-onprem --vnet-gateway1 vgw-azure --vnet-gateway2 vgw-onprem --shared-key "LabOnly-ChangeMe-123" -o none
az network vpn-connection create -g $RG -n onprem-to-azure --vnet-gateway1 vgw-onprem --vnet-gateway2 vgw-azure --shared-key "LabOnly-ChangeMe-123" -o none
az network vpn-connection show -g $RG -n azure-to-onprem --query connectionStatus

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** hybrid connectivity extends your trust boundary both ways. Filter traffic between on-premises and Azure at the firewall; do not assume "internal" means trusted.
- **Security:** `VirtualNetwork` in NSG rules includes on-premises prefixes once a gateway is connected.
- **Security:** use Entra authentication and Conditional Access for point-to-site. Rotate shared keys; prefer certificate or stronger policies for site-to-site.
- **Security:** ExpressRoute is not encrypted. Say so plainly when asked.
- **Cost:** gateways bill hourly whether used or not. ExpressRoute adds circuit, provider and possibly egress charges. Two circuits for resilience doubles the fixed cost; that is the price of the SLA.
- **Reliability:** a single circuit, a single peering location, a single on-premises device, or a single provider is a single point of failure. Find each and decide deliberately.
- **Reliability:** gateway maintenance happens. Active-active and zone-redundant SKUs make it invisible.

## Common mistakes

- One ExpressRoute circuit at one peering location for production.
- Treating a circuit's two links as primary and unused spare, and never testing the second.
- Assuming ExpressRoute is encrypted.
- `GatewaySubnet` too small, or with an NSG or a default-route UDR attached.
- Static routes where BGP would give automatic failover.
- Connectivity built without hybrid DNS.
- Basic or non-zonal gateway SKUs in production.
- Overlapping address spaces between on-premises and Azure.
- Asymmetric routing because the gateway subnet has no route to the firewall for spoke prefixes.
- Ordering a circuit the week it is needed.

## Check yourself

1. Give four differences between VPN Gateway and ExpressRoute.
2. What is the difference between private peering and Microsoft peering?
3. Is ExpressRoute traffic encrypted? What are the options?
4. Describe the three ExpressRoute resiliency levels.
5. What is the benefit of an active-active VPN gateway with BGP?
6. Why is hybrid DNS as important as the circuit itself?
7. What does FastPath do?
8. Where do the gateways sit in a landing zone, and how do spokes use them?

<details>
<summary>Answers</summary>

1. VPN runs over the internet and ExpressRoute over a private provider connection; VPN is encrypted by default and ExpressRoute is not; ExpressRoute offers higher, more predictable bandwidth and latency; VPN is quick and cheap to set up while ExpressRoute takes weeks and costs more.
2. Private peering reaches your virtual networks' private address space; Microsoft peering reaches Microsoft public service endpoints over the circuit.
3. Not by default. Options are application-layer TLS, MACsec on ExpressRoute Direct, or an IPsec VPN over private peering.
4. Standard: one circuit at one peering location. High: ExpressRoute Metro, with links to two peering locations in one metro. Maximum: two circuits at two different peering locations.
5. Both instances carry traffic and BGP reroutes automatically on failure, giving faster, hitless failover.
6. Private endpoints and internal services are reached by name; without forwarding to Azure DNS, on-premises clients resolve public addresses or nothing.
7. Sends data-plane traffic directly to VMs, bypassing the ExpressRoute gateway, for better performance.
8. In the hub VNet of the connectivity subscription. Spokes use them through peering with gateway transit enabled on the hub and remote gateways on the spoke.

</details>

## References

- [What is Azure VPN Gateway?](https://learn.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpngateways)
- [About VPN Gateway SKUs](https://learn.microsoft.com/azure/vpn-gateway/about-gateway-skus)
- [Design highly available gateway connectivity](https://learn.microsoft.com/azure/vpn-gateway/vpn-gateway-highlyavailable)
- [What is Azure ExpressRoute?](https://learn.microsoft.com/azure/expressroute/expressroute-introduction)
- [ExpressRoute circuits and peering](https://learn.microsoft.com/azure/expressroute/expressroute-circuit-peerings)
- [Design and architect ExpressRoute for resiliency](https://learn.microsoft.com/azure/expressroute/design-architecture-for-resiliency)
- [About ExpressRoute Metro](https://learn.microsoft.com/azure/expressroute/metro)
- [About ExpressRoute FastPath](https://learn.microsoft.com/azure/expressroute/about-fastpath)
- [About encryption for Azure ExpressRoute](https://learn.microsoft.com/azure/expressroute/expressroute-about-encryption)
- [Connect an on-premises network to Azure (Architecture Center)](https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/)
