# Azure DNS

## Why it matters

In a private Azure environment, DNS is the difference between a working system and a set of timeouts. Every private endpoint depends on it, hybrid connectivity depends on it, and it is the most frequent root cause when an isolated AI deployment "can't reach" its model or search service. This file covers Azure's DNS services and the resolution designs you need for landing zones and hybrid networks.

## Key concepts

### The DNS services

| Service | Purpose |
|---|---|
| **Azure DNS (public zones)** | Authoritative hosting for your internet domains |
| **Azure Private DNS (private zones)** | Name resolution inside virtual networks, without running DNS servers |
| **Azure DNS Private Resolver** | Managed conditional forwarding between Azure and on-premises |
| **Azure-provided DNS** | The default recursive resolver for every VNet, at `168.63.129.16` |
| **DNS security policy** | Filtering and logging of DNS queries from virtual networks |
| **Azure Traffic Manager** | DNS-based global traffic routing (file 07) |

Azure DNS hosts zones. It is not a domain registrar; buy domains elsewhere (or through App Service Domains) and delegate.

### Public zones

- Create a zone for `contoso.com`, then set the registrar's NS records to the four Azure name servers assigned to the zone.
- Record types: A, AAAA, CNAME, MX, TXT, SRV, CAA, NS, PTR, and others.
- **Alias records** point an A, AAAA or CNAME record set at an Azure resource (public IP, Front Door, Traffic Manager, Static Web App). They update automatically when the resource's address changes and allow an **apex** (`contoso.com`) record to target a service that would otherwise need a CNAME.
- **DNSSEC** signing is supported for public zones.
- SLA of 100% for valid requests; globally anycast.
- Manage with RBAC, locks and IaC. A deleted production zone is a serious outage.

### Private zones

A private zone is resolvable only from virtual networks **linked** to it.

| Link type | Behaviour |
|---|---|
| **Resolution link** | VMs in the VNet can resolve records in the zone |
| **Registration link** (auto-registration) | Additionally, VMs in the VNet automatically get A records in the zone. A VNet can be a registration VNet for one zone only |

Uses:

1. **Private endpoint zones** (`privatelink.*`), holding the records that map service names to private IPs.
2. **Your own internal names** (`corp.contoso.internal`) for VMs and services, with auto-registration.
3. **Split-horizon**: the same zone name in public and private DNS returning different answers inside and outside.

Limits to know: up to 1,000 VNet links per zone (higher with a feature), 25,000 record sets per zone.

**Fallback to internet.** A private zone linked to a VNet is authoritative for its name, so a query for a record that is **not** in it returns NXDOMAIN. That breaks access to resources of the same type that use public endpoints, for example a partner's storage account, once `privatelink.blob.core.windows.net` is linked. Setting the VNet link's resolution policy to **NxDomainRedirect** makes Azure fall back to public resolution when the private zone has no record. Enable it on `privatelink` zone links where you need to reach non-private resources of the same service.

### Azure-provided DNS: 168.63.129.16

A virtual IP present in every VNet. It:

- resolves public names recursively;
- resolves names in **private zones linked to that VNet**;
- resolves VM hostnames within the VNet;
- is reachable **only from inside Azure**. On-premises machines cannot query it.

That last point is the whole reason hybrid DNS needs extra components.

### VNet DNS settings

| Setting | Clients use |
|---|---|
| **Default (Azure-provided)** | `168.63.129.16` directly |
| **Custom DNS servers** | The IPs you list: a DNS Private Resolver inbound endpoint, Azure Firewall's DNS proxy, or DNS servers on VMs |

Changing the setting requires VMs to renew their lease or restart to pick it up.

### Azure DNS Private Resolver

A managed, zone-redundant service that replaces DNS forwarder VMs.

| Component | Role | Subnet |
|---|---|---|
| **Inbound endpoint** | A private IP that accepts queries from on-premises or other networks and resolves them using Azure DNS, including linked private zones | Dedicated, delegated, `/28` minimum |
| **Outbound endpoint** | The source of queries forwarded out of Azure | Dedicated, delegated, `/28` minimum |
| **DNS forwarding ruleset** | Rules mapping domain names to target DNS servers, linked to VNets | – |

```text
On-premises DNS                              Azure
  conditional forwarders:                    ┌──────────────────────────────────┐
   privatelink.* / azure zones ───────────►  │ Inbound endpoint 10.0.5.4        │──► Azure DNS ──► private zones
                                             │                                  │
  corp.contoso.com  ◄────────────────────────│ Outbound endpoint + ruleset:     │
   (authoritative on-premises)               │  corp.contoso.com → 192.168.1.10 │
                                             └──────────────────────────────────┘
```

- **On-premises → Azure:** on-premises DNS servers get conditional forwarders pointing at the inbound endpoint.
- **Azure → on-premises:** a forwarding ruleset linked to the VNets sends queries for on-premises domains through the outbound endpoint to on-premises DNS servers.

### Forwarding for private endpoints from on-premises

Forward the **public** zone name, not the `privatelink` name:

| Forward this | Not this |
|---|---|
| `blob.core.windows.net` | `privatelink.blob.core.windows.net` |
| `openai.azure.com`, `cognitiveservices.azure.com`, `services.ai.azure.com` | their `privatelink.*` equivalents |
| `search.windows.net` | `privatelink.search.windows.net` |
| `vaultcore.azure.net` | `privatelink.vaultcore.azure.net` |

Clients ask for `sthr.blob.core.windows.net`. The on-premises server must send that query to Azure, where the CNAME to `privatelink` is followed and the private zone answers. Forwarding only the `privatelink` name does not work reliably because the first lookup for the public name never reaches Azure.

The trade-off: forwarding `blob.core.windows.net` sends **all** blob lookups to Azure, which is fine since Azure DNS resolves the public ones too (and why the NXDOMAIN fallback above matters).

### Azure Firewall DNS proxy

Azure Firewall can act as a DNS proxy: clients point at the firewall's private IP, and it forwards to Azure DNS or custom servers. Benefits:

- **FQDN-based network rules** work reliably, because the firewall and the client resolve names the same way.
- One DNS address to configure across all spokes.
- DNS queries are logged.

A common landing zone chain: spoke VNets → custom DNS = firewall private IP → firewall DNS proxy → DNS Private Resolver or Azure DNS → private zones linked to the hub.

### DNS security policy

A policy linked to virtual networks that can **block, allow or alert** on queries by domain list, and **log** all DNS queries to Log Analytics, Storage or Event Hubs. Useful for blocking known-malicious or unwanted domains, and for the visibility DNS logs give during investigations. Threat-intelligence-based filtering is available as a managed feed.

## How it works on Azure

### Central DNS in a landing zone

```text
Connectivity subscription
├── Private DNS zones (all privatelink.* zones, once)
│      └── linked to: hub VNet  (resolution)
├── DNS Private Resolver in the hub
│      ├── inbound endpoint   ← on-premises conditional forwarders
│      └── outbound endpoint + ruleset → on-premises domains
└── Azure Firewall with DNS proxy (optional)

Spoke VNets
└── custom DNS = firewall IP (or resolver inbound endpoint)
     → every lookup is answered from the hub, where the zones are linked
```

Why this shape:

- **Zones exist once.** No conflicting copies, one place to look.
- **Spokes need no zone links** when they resolve through the hub, which keeps the design simple and well below link limits.
- **Policy does the wiring.** A `deployIfNotExists` policy adds a DNS zone group to each new private endpoint, registering it in the central zone.
- **On-premises resolves the same names** through the inbound endpoint.

An alternative for smaller estates: link every spoke directly to the zones and leave VNets on Azure-provided DNS. It works, with more links to manage and no hybrid resolution.

### Multi-region

Run a resolver (and firewall) in each regional hub. Private zones are global resources, so link them to each hub. On-premises forwarders list both regions' inbound endpoints for resilience.

### Resolution walk-through

A container app in a spoke calls `aif-hr.openai.azure.com`:

1. The app's resolver is the spoke VNet's DNS setting: the hub firewall.
2. The firewall's DNS proxy forwards to Azure DNS (`168.63.129.16`) from the hub.
3. Azure DNS resolves the public name, finds `CNAME aif-hr.privatelink.openai.azure.com`.
4. The hub VNet is linked to `privatelink.openai.azure.com`, which has an A record `10.20.3.6`.
5. The answer `10.20.3.6` returns to the app, which connects to the private endpoint.

If step 4 fails (zone not linked, record missing), the answer is the **public** IP, and with public access disabled the connection is refused. The symptom is a 403 or timeout, and the cause is DNS.

### Troubleshooting DNS

```bash
# What does this name resolve to, and through which CNAMEs?
nslookup aif-hr.openai.azure.com
dig +short aif-hr.openai.azure.com

# Which resolver am I using?
cat /etc/resolv.conf            # Linux      ipconfig /all  on Windows

# Ask Azure DNS directly (from inside Azure only)
nslookup aif-hr.openai.azure.com 168.63.129.16

# Is the record there, and is the zone linked?
az network private-dns record-set a list -g <dns-rg> -z privatelink.openai.azure.com -o table
az network private-dns link vnet list -g <dns-rg> -z privatelink.openai.azure.com \
  --query "[].{Link:name, VNet:virtualNetwork.id, Fallback:resolutionPolicy}" -o table

# What DNS servers does the VNet hand out?
az network vnet show -g <rg> -n <vnet> --query dhcpOptions.dnsServers
```

Diagnosis table:

| Result from inside the network | Likely cause |
|---|---|
| Public IP returned | Zone not linked to the resolving VNet; record missing; client using a resolver that bypasses Azure DNS |
| NXDOMAIN for a resource that should be public | `privatelink` zone linked without fallback, and the resource has no private endpoint here |
| Correct private IP, connection times out | Not DNS. Check NSG, routing, firewall |
| Works in Azure, fails on-premises | Conditional forwarder missing or pointing at the wrong name; inbound endpoint unreachable |
| Works after a while | Negative or stale cache; TTL |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Azure-provided DNS or custom | Azure-provided for simple, cloud-only networks. Custom (resolver or firewall proxy) for landing zones and hybrid |
| DNS Private Resolver or forwarder VMs | Private Resolver: managed, zone-redundant, no patching. VMs only for features it lacks |
| Central or per-spoke zone links | Central, with spokes resolving through the hub |
| Firewall DNS proxy | Yes when using FQDN network rules or wanting one DNS address and query logs |
| Fallback to internet on `privatelink` zones | Enable where workloads must reach non-private resources of the same service type |
| Own internal zone naming | Use a domain you own (a subdomain such as `az.contoso.com`), not a made-up TLD |
| Public DNS hosting | Azure DNS is a solid default; use alias records for Azure targets and protect zones with locks |

## Hands-on

Continues from file 05 (`rg-vnet-lab` with a storage private endpoint).

```bash
RG=rg-vnet-lab; VNET=vnet-lab-swc-001

# 1. A private zone for your own names, with auto-registration
az network private-dns zone create -g $RG -n lab.contoso.internal -o none
az network private-dns link vnet create -g $RG -z lab.contoso.internal -n reg-$VNET \
  --virtual-network $VNET --registration-enabled true -o none
az network private-dns record-set a add-record -g $RG -z lab.contoso.internal -n api -a 10.20.1.10 -o none
az network private-dns record-set list -g $RG -z lab.contoso.internal -o table

# 2. Inspect the privatelink zone and its link; enable fallback to internet
az network private-dns link vnet list -g $RG -z privatelink.blob.core.windows.net -o table
az network private-dns link vnet update -g $RG -z privatelink.blob.core.windows.net -n link-$VNET \
  --resolution-policy NxDomainRedirect -o none

# 3. DNS Private Resolver with an inbound endpoint
az network vnet subnet create -g $RG --vnet-name $VNET -n snet-dns-in --address-prefixes 10.20.3.128/28 \
  --delegations Microsoft.Network/dnsResolvers -o none
az extension add -n dns-resolver 2>/dev/null
VNET_ID=$(az network vnet show -g $RG -n $VNET --query id -o tsv)
az dns-resolver create -g $RG -n dnspr-lab -l swedencentral --id $VNET_ID -o none
az dns-resolver inbound-endpoint create -g $RG --dns-resolver-name dnspr-lab -n inbound -l swedencentral \
  --ip-configurations "[{private-ip-allocation-method:Dynamic,id:$VNET_ID/subnets/snet-dns-in}]" -o none
az dns-resolver inbound-endpoint show -g $RG --dns-resolver-name dnspr-lab -n inbound \
  --query "ipConfigurations[0].privateIpAddress"

# 4. A public zone (no cost to create; do not delegate a real domain)
az network dns zone create -g $RG -n lab-example-roadmap.com --query nameServers
```

The address from step 3 is what on-premises conditional forwarders would target. With a VM in the VNet, resolve the storage account name and `api.lab.contoso.internal`, then query the inbound endpoint IP directly.

Paper exercise: draw the resolution path for an on-premises laptop reaching a private Foundry endpoint, naming each hop and the forwarder rule that makes it work.

```bash
az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** DNS is a common exfiltration and command-and-control channel. Log queries (firewall DNS proxy or DNS security policy) and block known-bad domains.
- **Security:** protect zones with RBAC and locks. Whoever controls `privatelink` records controls where traffic for your services goes.
- **Security:** watch for dangling public CNAMEs pointing at deleted resources; they enable subdomain takeover. Alias records and Defender for DNS-style checks help.
- **Cost:** private zones are billed per zone and per million queries, at low rates. Private Resolver endpoints have an hourly charge each; a hub with inbound and outbound endpoints in two regions is a noticeable fixed cost.
- **Reliability:** every workload depends on the central resolver path. Make it zone-redundant and deploy per region.
- **Reliability:** keep TTLs modest on records you may need to change in a failover.

## Common mistakes

- Private endpoint zones not linked to the VNet where resolution happens.
- Forwarding `privatelink.*` names from on-premises instead of the public zone names.
- Pointing on-premises forwarders at `168.63.129.16`.
- Multiple copies of the same `privatelink` zone.
- Creating a private zone for a public suffix such as `blob.core.windows.net`.
- Linking a `privatelink` zone and breaking access to public resources of that service, with no fallback configured.
- Changing VNet DNS servers and not renewing leases on running VMs.
- A resolver subnet that is too small or not delegated.
- Treating a DNS problem as a firewall problem for hours.

## Check yourself

1. What is `168.63.129.16`, and who can query it?
2. What is the difference between a resolution link and a registration link?
3. Why do on-premises networks need a DNS Private Resolver inbound endpoint (or equivalent)?
4. Which zone name should on-premises DNS forward for private blob access?
5. A name resolves to a public IP from inside the spoke. Name two likely causes.
6. What problem does the NxDomainRedirect resolution policy solve?
7. Why enable the DNS proxy on Azure Firewall?
8. What is an alias record useful for?

<details>
<summary>Answers</summary>

1. The Azure-provided DNS virtual IP available in every VNet. Only resources inside Azure can query it.
2. A resolution link lets a VNet resolve records in the zone; a registration link also auto-registers the VNet's VMs in the zone.
3. On-premises servers cannot reach Azure-provided DNS, so they need a private IP in Azure to forward queries to, which then resolves using linked private zones.
4. `blob.core.windows.net`, not `privatelink.blob.core.windows.net`.
5. The private zone is not linked to the VNet doing the resolution; the A record is missing; or the client uses a resolver that bypasses Azure DNS.
6. Resources of the same service type without a private endpoint returning NXDOMAIN once the `privatelink` zone is linked; it falls back to public resolution.
7. So FQDN-based network rules resolve consistently, to give spokes one DNS address, and to log queries.
8. Pointing a record, including a zone apex, at an Azure resource so it tracks the resource's address automatically.

</details>

## References

- [What is Azure DNS?](https://learn.microsoft.com/azure/dns/dns-overview)
- [What is Azure Private DNS?](https://learn.microsoft.com/azure/dns/private-dns-overview)
- [What is a virtual network link?](https://learn.microsoft.com/azure/dns/private-dns-virtual-network-links)
- [What is Azure DNS Private Resolver?](https://learn.microsoft.com/azure/dns/dns-private-resolver-overview)
- [Resolve Azure and on-premises domains](https://learn.microsoft.com/azure/dns/private-resolver-hybrid-dns)
- [Azure Firewall DNS settings](https://learn.microsoft.com/azure/firewall/dns-settings)
- [DNS security policy](https://learn.microsoft.com/azure/dns/dns-security-policy)
- [Private Link and DNS integration at scale](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/private-link-and-dns-integration-at-scale)
- [What is IP address 168.63.129.16?](https://learn.microsoft.com/azure/virtual-network/what-is-ip-address-168-63-129-16)
- [Azure DNS alias records overview](https://learn.microsoft.com/azure/dns/dns-alias)
