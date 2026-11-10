# Network Security Groups and Application Security Groups

## Why it matters

Network security groups are Azure's built-in, free packet filter. They are the first control between subnets, the usual reason "it can't connect", and a baseline requirement in every security benchmark. They are simple to describe and easy to get subtly wrong, mostly through misunderstanding the default rules and evaluation order.

## Key concepts

### What an NSG is

A list of allow and deny rules that filter traffic by **5-tuple**: source, source port, destination, destination port and protocol. NSGs are **stateful**: if a connection is allowed in one direction, return traffic is allowed automatically.

An NSG can be associated with:

- a **subnet** (applies to everything in it), and/or
- a **network interface** (applies to that NIC).

One NSG can be attached to many subnets and NICs. Each subnet and NIC can have at most one NSG.

### Rule properties

| Property | Values |
|---|---|
| Priority | 100 to 4096. **Lower number wins.** Evaluation stops at the first match |
| Direction | Inbound or outbound |
| Source / destination | IP or CIDR, **service tag**, **application security group**, or `*` |
| Port ranges | Single, range, list, or `*` |
| Protocol | TCP, UDP, ICMP, ESP, AH, or Any |
| Action | Allow or Deny |

**Augmented rules** let one rule hold multiple addresses, ranges and ports, which keeps rule counts down.

### Default rules

Every NSG contains these. They cannot be deleted, only overridden by higher-priority rules.

**Inbound**

| Priority | Name | Source | Destination | Action |
|---|---|---|---|---|
| 65000 | AllowVnetInBound | VirtualNetwork | VirtualNetwork | Allow |
| 65001 | AllowAzureLoadBalancerInBound | AzureLoadBalancer | Any | Allow |
| 65500 | DenyAllInBound | Any | Any | Deny |

**Outbound**

| Priority | Name | Source | Destination | Action |
|---|---|---|---|---|
| 65000 | AllowVnetOutBound | VirtualNetwork | VirtualNetwork | Allow |
| 65001 | AllowInternetOutBound | Any | Internet | Allow |
| 65500 | DenyAllOutBound | Any | Any | Deny |

Three consequences people miss:

1. **Everything inside the virtual network is allowed by default.** The `VirtualNetwork` tag includes the VNet's own address space, **peered VNets**, and **on-premises ranges** learned through gateways. A fresh NSG does not segment subnets from each other.
2. **Outbound to the internet is allowed by default** at the NSG level. Whether packets actually get out depends on having an outbound path (file 11), but the NSG will not stop them.
3. **Inbound from the internet is denied** by default.

### Evaluation order

```text
Inbound:   subnet NSG  ──►  NIC NSG  ──►  VM         (both must allow)
Outbound:  NIC NSG     ──►  subnet NSG ──► network   (both must allow)
```

Traffic between two VMs **in the same subnet** is also filtered by the subnet NSG. An overly broad deny at subnet level will break intra-subnet communication.

If Azure Virtual Network Manager **security admin rules** exist, they are evaluated **before** NSGs (file 04).

### Service tags

Named, Microsoft-maintained groups of IP prefixes. They save you from tracking address ranges that change.

| Tag | Represents |
|---|---|
| `VirtualNetwork` | VNet, peered VNets and connected on-premises ranges |
| `Internet` | Everything outside the virtual network, including Azure public IPs |
| `AzureLoadBalancer` | Health probe source (`168.63.129.16`) |
| `AzureCloud`, `AzureCloud.<region>` | All Azure public IP ranges, optionally for one region |
| `Storage`, `Sql`, `AzureKeyVault`, `AzureCosmosDB`, `AzureContainerRegistry`, `EventHub`, `ServiceBus` (many with `.<region>` variants) | Service ranges |
| `AzureActiveDirectory` | Microsoft Entra ID |
| `AzureMonitor`, `AzureResourceManager`, `AzureFrontDoor.Backend`, `GatewayManager`, `ApiManagement`, `AppService` | Platform services |
| `CognitiveServicesManagement`, `CognitiveServicesFrontend`, `AzureMachineLearning`, `AzureOpenAI` | AI service ranges (check the current list for exact names and scope) |

A service tag covers the **whole service** for all customers, not just your resources. Allowing `Storage` outbound allows every storage account in Azure, which does not prevent exfiltration to an attacker's account. For that, use private endpoints and a firewall with FQDN rules.

### Application security groups

An ASG is a named group of NICs. Rules reference the group instead of IP addresses.

```text
asg-web   = NICs of web servers
asg-api   = NICs of API servers
asg-db    = NICs of database servers

Allow  Internet → asg-web   443
Allow  asg-web  → asg-api   8443
Allow  asg-api  → asg-db    1433
Deny   VirtualNetwork → VirtualNetwork  *      (priority 4000)
```

Rules now follow the workload's structure and survive IP changes and scale-out. Constraints: all NICs in an ASG must be in the same VNet, and when a rule uses ASGs for both source and destination they must be in the same VNet.

ASGs apply to NICs, so they suit VM-based tiers. PaaS integrations and private endpoints are handled with subnets and, for private endpoints, network policies.

### Private endpoints and NSGs

By default, NSGs and route tables **do not apply** to private endpoint traffic. Set the subnet property `privateEndpointNetworkPolicies` to `Enabled` (or `NetworkSecurityGroupEnabled` / `RouteTableEnabled`) to have them take effect. With it enabled you can restrict which sources reach each private endpoint, which is good practice in a shared spoke.

### What NSGs do not do

| Not provided | Use instead |
|---|---|
| FQDN or URL filtering | Azure Firewall |
| TLS inspection, IDPS | Azure Firewall Premium |
| Layer 7 protection (SQL injection, XSS) | Web Application Firewall |
| Filtering for PaaS services outside the VNet | Service firewalls, private endpoints, Network Security Perimeter |
| Central rules nobody can override | AVNM security admin rules, Azure Policy |

### Flow logging

| | **VNet flow logs** | **NSG flow logs** |
|---|---|---|
| Status | Current, recommended | **Retiring 30 September 2027**; new ones could not be created after 30 June 2025 |
| Scope | Virtual network, subnet or NIC | Per NSG |
| Covers | All traffic, including that evaluated by AVNM security admin rules; shows encryption status | Traffic evaluated by that NSG |

Flow logs are written to a storage account. **Traffic Analytics** processes them into a Log Analytics workspace for querying and visualisation: top talkers, blocked flows, traffic to malicious IPs, flows by region and port. Migrate any remaining NSG flow logs to VNet flow logs.

## How it works on Azure

### A baseline for a workload spoke

| Subnet | Inbound allow | Outbound allow | Then |
|---|---|---|---|
| `snet-appgw` | Internet → 443; `GatewayManager` → 65200–65535; `AzureLoadBalancer` → any | To `snet-apps` on app ports | Deny rest |
| `snet-apps` | From `snet-appgw` on app ports | To `snet-pe` 443; to hub firewall / required service tags | Deny other VNet traffic |
| `snet-pe` (policies enabled) | From `snet-apps`, `snet-agents`, `snet-mgmt` 443 | – | Deny rest |
| `snet-mgmt` | From Bastion subnet 22/3389 | To `snet-pe` 443, required egress | Deny rest |

Add an explicit low-priority **deny VirtualNetwork → VirtualNetwork** inbound rule (for example priority 4000) on each subnet NSG, above the default allow, and then open only what is needed. That turns the default-allow VNet into default-deny between tiers.

Platform-injected services publish **required** NSG rules (Application Gateway, API Management, Bastion, Container Apps, AKS, SQL Managed Instance). Blocking them breaks the service in ways that surface as health or management failures. Always check the service's networking page.

### Rule design practices

- **One NSG per subnet**, named for it. Use NIC-level NSGs only for exceptions.
- **Leave gaps** in priorities (100, 110, 120…) to insert rules later.
- **Prefer service tags and ASGs** to IP lists.
- **Never** allow `Internet → *` on 22 or 3389. Use Bastion or just-in-time access.
- **Manage as code.** NSG rules drift quickly when edited by hand.
- **Describe every rule.** The description field is free documentation.

### Central enforcement

- **Azure Policy:** *Subnets should have a Network Security Group*; deny rules that expose management ports to the internet.
- **AVNM security admin rules:** organisation-wide allow, deny or always-allow rules that workload NSGs cannot override.
- **Defender for Cloud:** recommendations for overly permissive rules and adaptive network hardening.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Subnet or NIC association | Subnet. NIC-level only for specific exceptions |
| NSG or Azure Firewall | Both, at different layers. NSGs for coarse segmentation between subnets; a firewall for inspection, FQDN filtering and egress control |
| Micro-segmentation depth | Deny between tiers by default in production. Finer segmentation costs operational effort; apply it where data sensitivity warrants |
| Service tags for egress | Acceptable for platform dependencies. Not an exfiltration control |
| Flow logs everywhere | VNet flow logs on production networks at least; sample or scope to control cost |

## Hands-on

Uses the VNet from file 01.

```bash
RG=rg-vnet-lab; VNET=vnet-lab-swc-001

# 1. Create an NSG and inspect the default rules
az network nsg create -g $RG -n nsg-snet-mgmt -o none
az network nsg show -g $RG -n nsg-snet-mgmt \
  --query "defaultSecurityRules[].{Pri:priority, Name:name, Dir:direction, Src:sourceAddressPrefix, Dst:destinationAddressPrefix, Action:access}" -o table

# 2. Default-deny between subnets, then allow what is needed
az network nsg rule create -g $RG --nsg-name nsg-snet-mgmt -n DenyVnetInbound --priority 4000 \
  --direction Inbound --access Deny --protocol '*' \
  --source-address-prefixes VirtualNetwork --destination-address-prefixes VirtualNetwork --destination-port-ranges '*' -o none
az network nsg rule create -g $RG --nsg-name nsg-snet-mgmt -n AllowSshFromBastion --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.20.0.0/26 --destination-address-prefixes '*' --destination-port-ranges 22 \
  --description "SSH from AzureBastionSubnet only" -o none

# 3. Associate with the subnet
az network vnet subnet update -g $RG --vnet-name $VNET -n snet-mgmt --network-security-group nsg-snet-mgmt -o none

# 4. Application security groups
az network asg create -g $RG -n asg-api -o none
az network nsg rule create -g $RG --nsg-name nsg-snet-mgmt -n AllowMgmtToApi --priority 200 \
  --direction Outbound --access Allow --protocol Tcp \
  --source-address-prefixes '*' --destination-asgs asg-api --destination-port-ranges 8443 -o none

# 5. List the effective custom rules
az network nsg rule list -g $RG --nsg-name nsg-snet-mgmt \
  --query "[].{Pri:priority, Name:name, Dir:direction, Action:access, Ports:destinationPortRange}" -o table

# 6. Look up what a service tag contains
az network list-service-tags -l swedencentral \
  --query "values[?name=='AzureKeyVault.SwedenCentral'].properties.addressPrefixes | [0][:5]"
```

With a VM in the subnet you can also run `az network nic list-effective-nsg` to see the merged result of subnet and NIC rules, and Network Watcher's *IP flow verify* (file 17).

Exercise: write the NSG rules for the four-subnet baseline table above.

## Security, cost and reliability notes

- **Security:** the default intra-VNet allow, including peered networks, is the gap attackers use for lateral movement. Close it deliberately.
- **Security:** NSGs see IPs and ports only. They cannot tell a legitimate HTTPS call from exfiltration over 443.
- **Cost:** NSGs and ASGs are free. Flow logs incur storage and, with Traffic Analytics, ingestion charges.
- **Reliability:** blocking `AzureLoadBalancer` or a platform service's management ports causes health-probe failures and broken updates. Keep required rules.
- **Limits:** up to 1,000 rules per NSG by default and 5,000 NSGs per subscription per region (adjustable); design with tags and ASGs to stay well below.

## Common mistakes

- Believing a new NSG isolates subnets.
- Forgetting that `VirtualNetwork` includes peered and on-premises ranges.
- Management ports open to `Internet` or `*`.
- Conflicting subnet-level and NIC-level NSGs that nobody can reason about.
- Assuming NSGs filter private endpoint traffic without enabling network policies.
- Treating service tags as a data-exfiltration control.
- Breaking a platform service by denying its required ports.
- Still depending on NSG flow logs.
- Long lists of hard-coded IPs instead of ASGs and tags.

## Check yourself

1. In what order are rules evaluated, and when does evaluation stop?
2. Name the three default inbound rules.
3. What does the `VirtualNetwork` service tag include?
4. For inbound traffic, which NSG is evaluated first: subnet or NIC?
5. What problem do application security groups solve?
6. How do you make NSG rules apply to private endpoint traffic?
7. Why is allowing the `Storage` service tag outbound not an exfiltration control?
8. What replaces NSG flow logs?

<details>
<summary>Answers</summary>

1. By priority, lowest number first; evaluation stops at the first matching rule.
2. AllowVnetInBound (65000), AllowAzureLoadBalancerInBound (65001), DenyAllInBound (65500).
3. The VNet's address space, peered VNets, and on-premises ranges connected through gateways.
4. The subnet NSG, then the NIC NSG.
5. They let rules reference groups of NICs by role instead of IP addresses, so rules follow the application structure and survive scaling and readdressing.
6. Enable private endpoint network policies on the subnet.
7. The tag covers every storage account in Azure, including ones an attacker controls.
8. Virtual network flow logs.

</details>

## References

- [Network security groups overview](https://learn.microsoft.com/azure/virtual-network/network-security-groups-overview)
- [How network security groups filter network traffic](https://learn.microsoft.com/azure/virtual-network/network-security-group-how-it-works)
- [Virtual network service tags](https://learn.microsoft.com/azure/virtual-network/service-tags-overview)
- [Application security groups](https://learn.microsoft.com/azure/virtual-network/application-security-groups)
- [Manage network policies for private endpoints](https://learn.microsoft.com/azure/private-link/disable-private-endpoint-network-policy)
- [Virtual network flow logs](https://learn.microsoft.com/azure/network-watcher/vnet-flow-logs-overview)
- [Traffic analytics](https://learn.microsoft.com/azure/network-watcher/traffic-analytics)
- [Azure best practices for network security](https://learn.microsoft.com/azure/security/fundamentals/network-best-practices)
