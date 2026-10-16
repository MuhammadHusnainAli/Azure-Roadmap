# Virtual Networks and Subnets

> Topic: 05 · Networking · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

The virtual network is the private boundary everything else attaches to. Address planning is one of the few decisions in Azure that is painful to change later: an overlapping range blocks peering and hybrid connectivity, and an undersized subnet blocks a deployment. AI workloads consume more subnets than people expect, because private endpoints, application integration and agent runtimes each want their own.

## Key concepts

### What a virtual network is

An isolated, software-defined network in **one region** and **one subscription**, defined by one or more address ranges and divided into **subnets**.

| Property | Detail |
|---|---|
| Scope | Regional. Spans all availability zones in the region |
| Isolation | No connectivity to other VNets unless you add it (peering, gateways) |
| Internal routing | All subnets in a VNet can reach each other by default |
| Cost | The VNet itself is free. You pay for what is in it and for traffic that crosses peerings, gateways and zones |
| Address space | One or more CIDR blocks; IPv4, optionally dual-stack with IPv6 |

### Address space

Use private ranges (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. The `100.64.0.0/10` shared range also works and is useful for address-hungry platforms.

Rules:

- **No overlaps** with any network you will ever connect: other VNets, on-premises, partner networks, other clouds.
- A VNet's address space can be **extended** later, and ranges can be added. Subnets can be resized only when the change does not strand existing addresses.
- Ranges cannot be shrunk below what subnets use.
- Avoid `172.17.0.0/16` in environments running Docker and `168.63.129.16/32` and `169.254.0.0/16`, which are reserved by the platform.

**Plan centrally.** In a landing zone the platform team allocates a non-overlapping block to each spoke from an IP address plan. Azure Virtual Network Manager's **IPAM** feature automates allocation from pools and tracks usage.

### Subnets

A subnet is a range inside the VNet. It is the unit to which you attach NSGs, route tables, NAT gateways, service endpoints and delegations.

**Azure reserves five addresses in every subnet:**

| Address | Use |
|---|---|
| `.0` | Network address |
| `.1` | Default gateway |
| `.2`, `.3` | Azure DNS mapping |
| Last | Broadcast |

So a `/29` (8 addresses) gives 3 usable, and a `/24` gives 251. `/29` is the smallest subnet allowed.

**Subnets with fixed names or minimum sizes**

| Subnet | Name | Recommended size |
|---|---|---|
| VPN / ExpressRoute gateway | `GatewaySubnet` | `/27` or larger |
| Azure Firewall | `AzureFirewallSubnet` | `/26` |
| Azure Firewall management (forced tunnelling, Basic SKU) | `AzureFirewallManagementSubnet` | `/26` |
| Azure Bastion | `AzureBastionSubnet` | `/26` or larger |
| Route Server | `RouteServerSubnet` | `/27` |
| Application Gateway | Any name, dedicated | `/24` recommended for v2 |
| DNS Private Resolver inbound / outbound | Any name, dedicated, delegated | `/28` minimum each |
| API Management (VNet injection or integration) | Dedicated | Depends on tier; `/27` or larger is common |
| Container Apps environment (workload profiles) | Dedicated, delegated | `/27` minimum; larger for scale |
| App Service / Functions VNet integration | Dedicated, delegated | `/26` or larger |
| AKS nodes | Dedicated | Depends on CNI mode and node count |
| Foundry Agent Service (network-isolated setup) | Dedicated, delegated | Check current guidance; `/24` is commonly recommended |
| Private endpoints | Any | Size for count: one IP each |

Requirements change; confirm in each service's networking documentation before fixing a design.

### Subnet delegation

Delegating a subnet to a service (for example `Microsoft.Web/serverFarms`, `Microsoft.App/environments`, `Microsoft.DBforPostgreSQL/flexibleServers`) lets that service inject and manage resources there. A delegated subnet is normally **exclusive** to that service, which is why each platform service wants its own.

### Private subnets and default outbound access

Historically a VM without any explicit outbound configuration still reached the internet through an Azure-owned, unpredictable IP. That **default outbound access was retired for new virtual networks on 31 March 2026**. In API versions released after that date, new subnets are **private by default** (`defaultOutboundAccess: false`): nothing in them can reach the internet until you provide an explicit path.

Explicit outbound methods: **NAT Gateway** (recommended), a **firewall or NVA** through a route, a **load balancer** outbound rule, or an **instance public IP**. Existing VNets that relied on default outbound continue to work, but new deployments must design egress deliberately. See file 11.

Private subnets also block some platform conveniences that silently depended on default outbound (for example Windows activation and updates without an explicit path), so plan for that.

### Service endpoints

A subnet setting that routes traffic for a PaaS service (Storage, SQL, Key Vault and others) over the Azure backbone and lets the service's firewall allow that subnet by identity.

| | Service endpoint | Private endpoint |
|---|---|---|
| Service address | Still the public IP | A private IP in your VNet |
| Reachable from on-premises | No | Yes, over VPN or ExpressRoute |
| Granularity | The whole service type in a region, restricted by the service firewall | One specific resource |
| DNS changes | None | Required |
| Data exfiltration protection | Limited (service endpoint policies for some services) | Strong: only that resource is reachable |
| Cost | Free | Hourly plus data charge |

Private endpoints are the default choice for new designs. Service endpoints remain useful where cost matters and on-premises reachability does not.

### Other VNet features

| Feature | Purpose |
|---|---|
| **Virtual network encryption** | Encrypts traffic between VMs within and across peered VNets, on supported VM sizes |
| **Accelerated networking** | SR-IOV on VM NICs for lower latency; on by default for most modern sizes |
| **IPv6 dual-stack** | Add IPv6 ranges alongside IPv4 |
| **Public IP prefixes / custom IP prefixes (BYOIP)** | Contiguous or self-owned public ranges |
| **Network Security Perimeter** | A logical boundary for PaaS services outside the VNet model (file 16) |
| **Service tags** | Named groups of Microsoft IP ranges for use in NSGs, routes and firewall rules |

### DNS setting on a VNet

A VNet uses **Azure-provided DNS** (`168.63.129.16`) by default, which resolves public names, names in linked private DNS zones, and VM names within the VNet. You can set custom DNS servers instead, typically the hub firewall's DNS proxy or a DNS Private Resolver. See file 06.

## How it works on Azure

### Planning a spoke for an AI workload

A `/22` (1,024 addresses) spoke for a private RAG and agent application:

| Subnet | Range | Purpose | Notes |
|---|---|---|---|
| `snet-appgw` | `10.20.0.0/24` | Application Gateway with WAF | Dedicated |
| `snet-apps` | `10.20.1.0/24` | Container Apps environment or App Service integration | Delegated |
| `snet-agents` | `10.20.2.0/24` | Foundry Agent Service injected compute | Delegated |
| `snet-pe` | `10.20.3.0/26` | Private endpoints (Foundry, Search, Storage, Cosmos DB, Key Vault, ACR) | NSG and route-table support enabled |
| `snet-apim` | `10.20.3.64/27` | API Management (if workload-local) | |
| `snet-mgmt` | `10.20.3.96/27` | Jump box or build agents | |
| spare | `10.20.3.128/25` | Growth | Leave room |

Principles:

- **One subnet per role.** It keeps NSGs and routes simple and satisfies delegation rules.
- **Leave space.** Unallocated ranges in each spoke cost nothing and save a re-address later.
- **Size for scale-out.** App platforms consume addresses per instance and during upgrades; follow each service's sizing formula.
- **Private endpoints share one subnet** unless you have a reason to separate them.
- **Same region as the workload.** VNets do not span regions; a second region is a second VNet with a non-overlapping range.

### Hub and spoke address plan

```text
10.0.0.0/16     Hub, region 1          (connectivity subscription)
10.1.0.0/16     Hub, region 2
10.10.0.0/16 …  Spokes, region 1       (allocated /22–/24 per workload)
10.50.0.0/16 …  Spokes, region 2
10.200.0.0/16   Sandbox (not routed to corporate network)
192.168.0.0/16, 172.16.0.0/12   On-premises (existing)
```

Summarisable blocks per region make routing and firewall rules far simpler.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| One large VNet or many | A VNet per workload per environment (spokes), connected through a hub. Avoid one shared VNet for everything |
| Spoke size | Generous. Running out of addresses is costly; unused private addresses are free. Balance against total organisational address space |
| Shared or dedicated subnets | Dedicated per platform service and per role |
| Service or private endpoints | Private endpoints for sensitive data, on-premises reachability and strict exfiltration control |
| IPv6 | Only for a concrete requirement. It adds design work across firewalls and services |
| Address scarcity | Where RFC 1918 space is exhausted, consider `100.64.0.0/10`, overlay networking for AKS, and NAT between overlapping networks |
| Who owns the VNet | The platform team provisions spokes through subscription vending; workload teams manage subnets within guardrails |

## Hands-on

```bash
RG=rg-vnet-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. VNet and subnets
az network vnet create -g $RG -n vnet-lab-swc-001 --address-prefixes 10.20.0.0/22 -o none
az network vnet subnet create -g $RG --vnet-name vnet-lab-swc-001 -n snet-apps --address-prefixes 10.20.1.0/24 \
  --delegations Microsoft.App/environments --default-outbound false -o none
az network vnet subnet create -g $RG --vnet-name vnet-lab-swc-001 -n snet-pe   --address-prefixes 10.20.3.0/26 \
  --private-endpoint-network-policies Enabled --default-outbound false -o none
az network vnet subnet create -g $RG --vnet-name vnet-lab-swc-001 -n snet-mgmt --address-prefixes 10.20.3.96/27 \
  --default-outbound false -o none

# 2. Inspect
az network vnet subnet list -g $RG --vnet-name vnet-lab-swc-001 \
  --query "[].{Name:name, Prefix:addressPrefix, Delegation:delegations[0].serviceName, DefaultOutbound:defaultOutboundAccess}" -o table

# 3. Usage: how many addresses are free?
az network vnet list-available-ips -g $RG -n vnet-lab-swc-001 | head -5
az network vnet subnet list-available-ips -g $RG --vnet-name vnet-lab-swc-001 -n snet-pe | head -3

# 4. Try an overlapping subnet and read the error
az network vnet subnet create -g $RG --vnet-name vnet-lab-swc-001 -n snet-bad --address-prefixes 10.20.1.128/25

# 5. Extend the address space
az network vnet update -g $RG -n vnet-lab-swc-001 --address-prefixes 10.20.0.0/22 10.21.0.0/24 -o none

# Keep this resource group for the next files, or delete:
# az group delete -n $RG --yes --no-wait
```

Paper exercise: you are given `10.30.0.0/23` for a workload that needs an Application Gateway, a Container Apps environment, eight private endpoints, an agent subnet and a jump box. Produce a subnet plan and say what does not fit.

## Security, cost and reliability notes

- **Security:** a VNet is a boundary, not a policy. Subnets inside it can talk freely until NSGs say otherwise.
- **Security:** treat private subnets as the norm. Explicit egress is easier to audit and control.
- **Cost:** traffic between availability zones and across peerings is billed per GB. Inter-region traffic costs more.
- **Reliability:** a VNet spans zones automatically. Zone resilience is a property of what you deploy into it.
- **Reliability:** subnet exhaustion causes scale-out and upgrade failures that look like application problems. Monitor available addresses.

## Common mistakes

- Overlapping address ranges discovered when peering or connecting on-premises.
- Subnets sized for today's instance count.
- Forgetting the five reserved addresses.
- Putting two delegated services in one subnet.
- Wrong name or size for `GatewaySubnet`, `AzureFirewallSubnet` or `AzureBastionSubnet`.
- Expecting internet access from a VM in a new private subnet with no NAT gateway.
- One flat VNet shared by every workload.
- Using every address in the block, leaving no room to grow.

## Check yourself

1. How many usable addresses are in a `/27` subnet in Azure?
2. Why must VNet address ranges not overlap with on-premises ranges?
3. What does subnet delegation do?
4. What changed about default outbound access in 2026, and what do you need instead?
5. Give three differences between service endpoints and private endpoints.
6. Which subnets require a specific name?
7. Why does an AI workload typically need several subnets?

<details>
<summary>Answers</summary>

1. 27 (32 minus the 5 reserved).
2. Overlaps prevent peering and make routing between the networks ambiguous or impossible.
3. It dedicates the subnet to a specific service so the service can inject and manage resources in it.
4. New subnets in new VNets are private by default, with no implicit internet access. You need an explicit method such as a NAT gateway, firewall, load balancer outbound rule or public IP.
5. Private endpoints give the service a private IP, are reachable from on-premises, target a single resource and need DNS changes; service endpoints keep the public IP, work only from the VNet, cover the service type and are free.
6. `GatewaySubnet`, `AzureFirewallSubnet`, `AzureFirewallManagementSubnet`, `AzureBastionSubnet`, `RouteServerSubnet`.
7. Ingress, application hosting, agent runtime, private endpoints and management each need a dedicated, often delegated, subnet with different rules.

</details>

## References

- [What is Azure Virtual Network?](https://learn.microsoft.com/azure/virtual-network/virtual-networks-overview)
- [Virtual network concepts and best practices](https://learn.microsoft.com/azure/virtual-network/concepts-and-best-practices)
- [Plan virtual networks](https://learn.microsoft.com/azure/virtual-network/virtual-network-vnet-plan-design-arm)
- [Add, change, or delete a virtual network subnet](https://learn.microsoft.com/azure/virtual-network/virtual-network-manage-subnet)
- [Default outbound access in Azure](https://learn.microsoft.com/azure/virtual-network/ip-services/default-outbound-access)
- [Virtual network service endpoints](https://learn.microsoft.com/azure/virtual-network/virtual-network-service-endpoints-overview)
- [Plan for IP addressing (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-ip-addressing)
- [IP address management in Azure Virtual Network Manager](https://learn.microsoft.com/azure/virtual-network-manager/concept-ip-address-management)
