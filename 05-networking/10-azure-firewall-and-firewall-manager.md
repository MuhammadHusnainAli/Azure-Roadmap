# Azure Firewall and Firewall Manager

## Why it matters

NSGs filter by IP and port. Azure Firewall is the managed, stateful firewall that filters by **domain name**, inspects traffic, and gives one controlled, logged exit for a whole landing zone. For AI workloads it answers a question security teams always ask: "what can this agent or application reach on the internet, and how do we know?"

## Key concepts

### What it is

A cloud-native firewall-as-a-service deployed into a hub virtual network (or a Virtual WAN hub). It is fully stateful, highly available by design, scales automatically, and has no instances for you to patch.

### SKUs

| | **Basic** | **Standard** | **Premium** |
|---|---|---|---|
| Intended for | Small environments | Most workloads | Regulated and high-security workloads |
| Throughput | Low, fixed scale | High, autoscaling | Highest, autoscaling |
| Network and application (FQDN) rules, DNAT | Yes | Yes | Yes |
| Threat intelligence | Alert only | Alert and deny | Alert and deny |
| DNS proxy, custom DNS | No | Yes | Yes |
| FQDN in network rules, web categories | No | Yes | Yes (more precise categories) |
| **TLS inspection** | No | No | Yes |
| **IDPS** (signature-based intrusion detection and prevention) | No | No | Yes |
| **URL filtering** (full path) | No | No | Yes |

You can change between Standard and Premium in place. Choose Premium when you need to see inside encrypted traffic or require IDPS.

### Rule types

| Rule type | Filters on | Use for |
|---|---|---|
| **DNAT** | Destination public IP and port → translated private IP and port | Publishing an internal service inbound (non-HTTP; use a WAF for HTTP) |
| **Network** | Source, destination IP / service tag / FQDN, port, protocol (TCP, UDP, ICMP, any) | Non-HTTP traffic and IP-based rules |
| **Application** | Source, destination **FQDN** (with wildcards), protocol HTTP/HTTPS/MSSQL; FQDN tags; web categories; URLs (Premium) | Outbound web traffic by domain name |

**Processing order:** DNAT rules, then network rules, then application rules. Within a policy, by rule collection group priority, then rule collection priority. The first match wins, and traffic matching nothing is **denied**.

A subtle consequence: if a network rule allows TCP 443 to `*`, application rules are never consulted for that traffic. Keep broad network allows out of the way of FQDN filtering.

### How FQDN filtering works

- For **HTTP**, the firewall reads the `Host` header.
- For **HTTPS**, it reads the **SNI** (server name indication) in the TLS handshake, without decrypting.
- With **TLS inspection** (Premium), it decrypts, inspects the full URL and content, and re-encrypts using a certificate authority you provide in Key Vault. Clients must trust that CA.
- **FQDN in network rules** resolves the name to IPs through DNS, so the firewall's **DNS proxy** should be enabled and clients should use it, ensuring both resolve identically.

**FQDN tags** are Microsoft-maintained groups of names for well-known services (`WindowsUpdate`, `AzureKubernetesService`, `AzureBackup`, and others). **Service tags** work in network rules for IP-based Microsoft ranges. **Web categories** allow or deny classes of site.

### Firewall policy

A **firewall policy** is a separate resource holding rules and settings. Firewalls are associated with a policy.

```text
Firewall policy
├── Settings: threat intelligence mode, DNS proxy, TLS inspection, IDPS, SNAT ranges
└── Rule collection groups   (priority)
     └── Rule collections    (type + action + priority)
          └── Rules
```

**Hierarchy:** a **child** policy inherits from a **parent**. Central security defines baseline rules in the parent (always evaluated first); regional or business-unit teams add rules in children. One policy can serve many firewalls across regions.

Other policy features: **IP groups** (reusable address sets), **draft and deploy** for staging changes, **policy analytics** for finding unused and overlapping rules, and **explicit proxy** mode for clients configured with a proxy address.

### Threat intelligence and IDPS

- **Threat intelligence-based filtering** alerts on or denies traffic to and from known-malicious IPs and domains from Microsoft's feed. It is evaluated before other rules.
- **IDPS** (Premium) matches traffic against tens of thousands of signatures, in *alert* or *alert and deny* mode, with per-signature overrides and bypass lists. To inspect HTTPS traffic it relies on TLS inspection.

### Deployment requirements

| Item | Detail |
|---|---|
| Subnet | `AzureFirewallSubnet`, `/26` |
| Management subnet | `AzureFirewallManagementSubnet` `/26`, required for Basic and for forced tunnelling; recommended generally so management traffic is separate |
| Public IPs | One or more Standard public IPs, or a NAT gateway on the firewall subnet for outbound scale |
| Availability zones | Deploy across zones for a higher SLA |
| Routing | Spoke subnets need a UDR sending `0.0.0.0/0` (and other-spoke prefixes) to the firewall's private IP |

Outbound traffic is **SNATed** to the firewall's public IP (or the NAT gateway's). Traffic to private ranges is not SNATed by default; this is configurable.

For very high outbound connection volumes, attaching a **NAT gateway** to the firewall subnet greatly increases available SNAT ports.

### Firewall Manager

A central management plane for:

- **Firewall policies** and their hierarchy across subscriptions and regions.
- **Secured virtual hubs**: Azure Firewall (or a partner security provider) in a Virtual WAN hub, with routing configured for you.
- **Hub virtual networks**: Azure Firewall in ordinary hub-and-spoke.
- **DDoS protection plans** and **WAF policies**, viewed and associated in one place.

### Alternatives

| Option | When |
|---|---|
| **Third-party NVAs** (Palo Alto, Fortinet, Check Point, and others) | An existing standard, a specific feature, or one vendor across clouds and on-premises. You operate the VMs, or use the vendor's managed offering |
| **Cloud NGFW as a SaaS in a Virtual WAN hub** | Vendor firewall, managed, in a secured hub |
| **NAT Gateway + NSGs only** | Small, low-risk environments needing predictable outbound IPs but no filtering |
| **Microsoft Entra Internet Access** | User-to-internet filtering by identity (topic 02), not workload egress |

## How it works on Azure

### Hub-and-spoke with Azure Firewall

```text
Spoke subnets ──UDR 0.0.0.0/0──► Azure Firewall (hub) ──► Internet
                                      │   └─► other spokes (inspected east-west)
On-premises ◄── gateway ──────────────┘   (GatewaySubnet routes spoke prefixes to the firewall)
```

| Traffic | Path | Rule type |
|---|---|---|
| Spoke → internet (egress) | Spoke UDR → firewall → SNAT → internet | Application rules by FQDN; network rules for non-HTTP |
| Spoke → spoke (east-west) | UDR → firewall → peer spoke | Network rules |
| On-premises ↔ spoke | Gateway ↔ firewall ↔ spoke | Network rules |
| Internet → internal non-HTTP service | Firewall public IP → DNAT → private IP | DNAT rule |
| Internet → web app | Application Gateway or Front Door (WAF), not the firewall | – |

**Firewall and Application Gateway together.** For inbound web traffic the usual pattern is the WAF first and direct to the backend, with the firewall handling egress. Where policy requires the firewall to see inbound web traffic too, place it after the Application Gateway with careful routing, and use Premium if it must inspect the re-encrypted traffic.

### Egress rules for an AI workload

A deny-by-default egress policy for a spoke hosting a Foundry application might allow:

| Purpose | Rule |
|---|---|
| Pull container images | Application rule: `mcr.microsoft.com`, `*.data.mcr.microsoft.com`, your registry (or reach ACR by private endpoint and skip this) |
| Entra ID sign-in and tokens | Service tag `AzureActiveDirectory`, or FQDNs `login.microsoftonline.com` and related |
| Azure Monitor telemetry | Service tag `AzureMonitor` (or Azure Monitor Private Link Scope) |
| Platform requirements of the compute host | The FQDN tags or documented names for Container Apps, AKS or App Service |
| Package feeds during build (build agents only) | `pypi.org`, `files.pythonhosted.org`, `registry.npmjs.org`, or better an internal artefact feed |
| Agent tools that reach the web | The specific FQDNs for each approved tool or MCP server, nothing wider |
| Web search grounding | The documented endpoints for the grounding service in use |
| Everything else | Denied and logged |

Foundry, AI Search, Storage, Cosmos DB and Key Vault are reached through **private endpoints**, so they need no firewall rule and their traffic does not leave the virtual network.

This list is the practical answer to data exfiltration by a manipulated agent: it can only talk to names you approved. Start with the firewall in a learning posture (allow with logging in a non-production environment), read the logs to see what the workload contacts, then write the allow-list and deny the rest.

### Logging

Enable diagnostic settings with **resource-specific (structured) logs** to Log Analytics.

| Table | Contains |
|---|---|
| `AZFWApplicationRule` | FQDN-level allows and denies |
| `AZFWNetworkRule` | Network rule hits |
| `AZFWNatRule` | DNAT translations |
| `AZFWDnsQuery` | DNS proxy queries |
| `AZFWThreatIntel`, `AZFWIdpsSignature` | Threat intelligence and IDPS events |
| `AZFWFlowTrace`, fat-flow and top-flow logs | Troubleshooting |

```kusto
// What is being denied, and to where?
AZFWApplicationRule
| where TimeGenerated > ago(24h) and Action == "Deny"
| summarize hits = count() by SourceIp, Fqdn, Protocol
| order by hits desc

// Every destination a subnet talks to (for building an allow-list)
AZFWApplicationRule
| where TimeGenerated > ago(7d) and SourceIp startswith "10.20.1."
| summarize count() by Fqdn, Action
```

The **Azure Firewall workbook** and **Policy Analytics** give ready-made views.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Standard or Premium | Premium when you must inspect encrypted traffic, need IDPS, or need URL-path filtering. Standard otherwise |
| TLS inspection | Powerful and intrusive. It needs CA distribution, breaks certificate-pinned clients, has privacy implications and costs throughput. Apply selectively and exclude sensitive categories |
| Azure Firewall or NVA | Azure Firewall for a managed, native default. NVAs for existing vendor standards or specific features |
| One firewall per region or shared | One per regional hub. Cross-region traffic through a single firewall adds latency and a regional dependency |
| Inspect east-west | Between workloads and trust zones, yes. Within a workload, usually NSGs suffice |
| Central or delegated rules | Parent policy for the baseline; child policies or rule collection groups per workload, managed as code by the owning team through pull requests |
| FQDN rules or service tags | FQDN application rules for precise control. Service tags for platform dependencies where names are impractical |

**Cost.** Azure Firewall has a significant fixed hourly charge per deployment plus a per-GB processing charge. Premium is the most expensive; Basic exists for small environments. In a small organisation the firewall is often the single largest line in the platform bill, which is why it is shared in a hub.

## Hands-on

Deploying a firewall for a short lab is relatively expensive; an hour or two is fine, a forgotten one is not. If you would rather not, do the review exercises.

**Deploy (optional)**

```bash
RG=rg-afw-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none
az extension add -n azure-firewall 2>/dev/null

az network vnet create -g $RG -n vnet-hub --address-prefixes 10.0.0.0/16 \
  --subnet-name AzureFirewallSubnet --subnet-prefixes 10.0.1.0/26 -o none
az network vnet subnet create -g $RG --vnet-name vnet-hub -n snet-workload --address-prefixes 10.0.10.0/24 -o none
az network public-ip create -g $RG -n pip-afw --sku Standard --zone 1 2 3 -o none

az network firewall policy create -g $RG -n afwp-lab --sku Standard --threat-intel-mode Deny \
  --enable-dns-proxy true -o none
az network firewall policy rule-collection-group create -g $RG --policy-name afwp-lab -n rcg-workloads --priority 200 -o none
az network firewall policy rule-collection-group collection add-filter-collection -g $RG --policy-name afwp-lab \
  --rule-collection-group-name rcg-workloads --name allow-web --collection-priority 100 --action Allow \
  --rule-type ApplicationRule --rule-name microsoft-docs \
  --source-addresses 10.0.10.0/24 --protocols Https=443 --target-fqdns learn.microsoft.com "*.github.com" -o none

az network firewall create -g $RG -n afw-lab --sku AZFW_VNet --tier Standard --zones 1 2 3 \
  --vnet-name vnet-hub --public-ip pip-afw --firewall-policy afwp-lab -o none
FWIP=$(az network firewall show -g $RG -n afw-lab --query "ipConfigurations[0].privateIPAddress" -o tsv)

az network route-table create -g $RG -n rt-workload --disable-bgp-route-propagation true -o none
az network route-table route create -g $RG --route-table-name rt-workload -n default \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWIP -o none
az network vnet subnet update -g $RG --vnet-name vnet-hub -n snet-workload --route-table rt-workload -o none
```

With a VM in `snet-workload` (its DNS set to the firewall IP): `curl -I https://learn.microsoft.com` succeeds; `curl -I https://www.example.com` fails. Enable diagnostics and find both in `AZFWApplicationRule`.

```bash
az group delete -n $RG --yes --no-wait
```

**Review exercises**

1. Write the egress allow-list, as application and network rules, for a container app that calls a private Foundry endpoint, pulls images from a private registry, sends telemetry to Application Insights, and has one agent tool that calls `api.weather.example.com`.
2. A network rule allows `10.20.0.0/16 → * TCP 443`. An application rule denies `*.dropbox.com`. A workload reaches Dropbox. Explain why.
3. Decide between Standard and Premium for a bank's internal document assistant and justify it.

<details>
<summary>Notes on the exercises</summary>

1. Application rules: registry and image hosts only if not private; `login.microsoftonline.com` (or the `AzureActiveDirectory` service tag as a network rule); `api.weather.example.com`. Network rule or private link scope for `AzureMonitor`. Platform FQDNs for Container Apps. Nothing for Foundry, since it is reached by private endpoint.
2. Network rules are processed before application rules. The broad 443 allow matches first, so the application deny is never evaluated.
3. Premium is defensible: IDPS and TLS inspection for regulated data and evidence of control over encrypted egress. Standard could suffice if all sensitive services are private-endpoint only and egress is tightly allow-listed by FQDN.

</details>

## Security, cost and reliability notes

- **Security:** SNI-based filtering can be evaded by a client that lies about the server name or uses encrypted client hello; TLS inspection and tight allow-lists reduce the gap.
- **Security:** wildcard FQDN rules such as `*.blob.core.windows.net` allow any storage account, including an attacker's. Prefer exact names or private endpoints.
- **Security:** protect the firewall policy and spoke route tables from change by workload teams.
- **Cost:** fixed cost plus data processing. Send only traffic that needs inspection; private endpoint and intra-spoke traffic normally bypass it.
- **Reliability:** deploy across availability zones. Scale-out takes several minutes, so pre-scale (minimum capacity settings where available) ahead of known peaks.
- **Reliability:** watch **SNAT port utilisation**; exhaustion causes intermittent outbound failures. Add public IPs or a NAT gateway.

## Common mistakes

- A broad network allow that pre-empts application rules.
- No UDR on spokes, so traffic never reaches the firewall.
- Missing `GatewaySubnet` routes, causing asymmetric hybrid traffic.
- DNS proxy off while using FQDN network rules.
- Publishing web applications through DNAT instead of a WAF.
- TLS inspection enabled globally, breaking pinned clients and platform services.
- Wildcard rules that defeat the purpose of filtering.
- No diagnostic logs, or legacy unstructured logs nobody can query.
- Treating the firewall as protection for private endpoint traffic that never passes through it.
- Forgetting the firewall is running in a lab subscription.

## Check yourself

1. What can Azure Firewall filter on that an NSG cannot?
2. In what order are DNAT, network and application rules processed?
3. How does the firewall filter HTTPS by FQDN without decrypting?
4. Name three Premium-only capabilities.
5. What is the purpose of a parent and child firewall policy?
6. Why do Foundry and AI Search usually need no egress rule in an isolated design?
7. Why enable the DNS proxy?
8. What symptom does SNAT port exhaustion cause, and how do you address it?

<details>
<summary>Answers</summary>

1. Fully qualified domain names, URLs and web categories, plus threat intelligence, IDPS and TLS-inspected content.
2. DNAT, then network, then application.
3. By reading the server name indication in the TLS handshake.
4. TLS inspection, IDPS, URL (path) filtering.
5. The parent holds centrally managed baseline rules evaluated first; children inherit them and add local rules.
6. They are reached through private endpoints inside the virtual network, so the traffic does not go to the internet via the firewall.
7. So the firewall and clients resolve names identically, making FQDN network rules reliable, and so DNS queries are logged.
8. Intermittent outbound connection failures. Add public IPs to the firewall or attach a NAT gateway to the firewall subnet.

</details>

## References

- [What is Azure Firewall?](https://learn.microsoft.com/azure/firewall/overview)
- [Choose the right Azure Firewall SKU](https://learn.microsoft.com/azure/firewall/choose-firewall-sku)
- [Azure Firewall rule processing logic](https://learn.microsoft.com/azure/firewall/rule-processing)
- [Azure Firewall Premium features](https://learn.microsoft.com/azure/firewall/premium-features)
- [Azure Firewall policy rule sets](https://learn.microsoft.com/azure/firewall/policy-rule-sets)
- [Azure Firewall DNS settings](https://learn.microsoft.com/azure/firewall/dns-settings)
- [Azure structured firewall logs](https://learn.microsoft.com/azure/firewall/firewall-structured-logs)
- [What is Azure Firewall Manager?](https://learn.microsoft.com/azure/firewall-manager/overview)
- [Firewall and Application Gateway for virtual networks](https://learn.microsoft.com/azure/architecture/example-scenario/gateway/firewall-application-gateway)
- [Scale SNAT ports with Azure NAT Gateway](https://learn.microsoft.com/azure/firewall/integrate-with-nat-gateway)
