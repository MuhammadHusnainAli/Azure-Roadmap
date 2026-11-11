# Network Security Design

## Why it matters

Identity is the primary control plane in Zero Trust, but the network still decides what an attacker can reach after compromising an identity, a VM or a container, and whether stolen data can leave. Topic 05 covered each networking service in depth; this file is about **designing** network security as a coherent architecture: segmentation, inspection, private access to PaaS, controlled ingress and egress, and monitoring. Architects are expected to justify where traffic is inspected, which paths are private, how egress is restricted and how the design limits lateral movement, all without making the platform impossible to operate.

## Key concepts

### Design principles (Well-Architected security)

| Principle | Meaning |
|---|---|
| **Segment** | Divide the estate into zones by workload, environment and sensitivity; deny by default between them |
| **Inspect** | Route traffic crossing trust boundaries through inspection (firewall, WAF, IDPS) |
| **Private by default** | PaaS through private endpoints; disable public network access; no default outbound internet |
| **Control egress** | Explicit, allow-listed outbound paths; prevent data exfiltration and command-and-control |
| **Protect ingress** | Public endpoints only through WAF and DDoS protection |
| **Secure administration** | No public management ports; Bastion, JIT, Entra-authenticated access |
| **Monitor** | Flow logs, firewall logs, WAF logs, DNS logs into Sentinel |

### Segmentation layers

| Layer | Controls | Topic 05 |
|---|---|---|
| **Macro** (environments, workloads) | Separate subscriptions and VNets; hub-and-spoke or Virtual WAN; no direct spoke-to-spoke without inspection | [04](../05-networking/04-vnet-peering-and-azure-virtual-network-manager.md), [13](../05-networking/13-virtual-wan.md) |
| **Micro** (tiers within a workload) | Subnets with NSGs and application security groups; Azure Virtual Network Manager security admin rules for org-wide guardrails | [02](../05-networking/02-network-security-groups-and-application-security-groups.md) |
| **PaaS perimeter** | Private endpoints, network security perimeter for PaaS-to-PaaS boundaries | [05](../05-networking/05-private-link-and-private-endpoints.md), [16](../05-networking/16-network-security-perimeter.md) |

### Inspection points

| Traffic | Inspection |
|---|---|
| **Inbound HTTP(S)** | Front Door WAF (global) or Application Gateway WAF (regional) ([Topic 05 files 08–09](../05-networking/08-application-gateway-and-web-application-firewall.md)) |
| **East-west between spokes** | Azure Firewall (Premium for TLS inspection and IDPS) in the hub via UDRs or routing intent |
| **Egress to internet** | Azure Firewall with FQDN/application rules, threat intelligence filtering; NAT Gateway for SNAT scale behind it where appropriate |
| **Hybrid** | Firewall between on-premises and Azure; ExpressRoute/VPN with encryption |
| **DNS** | Azure DNS Private Resolver; DNS security policies and logging to detect tunnelling/C2 |

### Egress control

Since **31 March 2026**, new subnets have no default outbound internet access (private subnets), which forces explicit egress design ([Topic 05 file 11](../05-networking/11-nat-gateway-and-outbound-connectivity.md)). Good egress design:

- route `0.0.0.0/0` from spokes to Azure Firewall;
- allow only required FQDNs per workload (package repositories, partner APIs, Microsoft endpoints via service tags/FQDN tags);
- use private endpoints so data services don't need internet paths;
- block or tightly restrict egress from data and AI subnets to reduce exfiltration risk.

### User and admin access

| Need | Control |
|---|---|
| Admin access to VMs | Azure Bastion, Defender for Servers JIT; no public RDP/SSH ([Topic 05 file 14](../05-networking/14-azure-bastion-and-secure-admin-access.md)) |
| User access to private apps | Microsoft Entra Private Access (Global Secure Access) instead of broad VPN |
| Internet access for users | Entra Internet Access / secure web gateway |

### DDoS

Azure provides infrastructure DDoS protection by default; **DDoS Network Protection** or **IP Protection** adds adaptive tuning, telemetry, rapid response and cost protection for public IPs ([Topic 05 file 15](../05-networking/15-ddos-protection.md)).

## How it works on Azure

Reference design (hub-and-spoke):

```text
Internet ──► Front Door (WAF, DDoS) ──► Private Link origin / App Gateway WAF in spoke
                                                │
On-premises ──ExpressRoute/VPN──► Hub VNet: Azure Firewall Premium (IDPS, TLS inspection, FQDN rules),
                                   Bastion, DNS Private Resolver, gateways
                                        │ UDR 0.0.0.0/0 and spoke-to-spoke via firewall
      ┌─────────────────────────────────┼─────────────────────────────────┐
  Spoke: web (NSG, ASG)        Spoke: data (PEs for SQL, Storage,      Spoke: AI (PEs for Foundry,
                               Key Vault; no internet egress)           Search; egress allow-list)
Azure Virtual Network Manager security admin rules: deny RDP/SSH from internet across all VNets
Azure Policy: deny public IPs on NICs, require private endpoints, deny public network access on PaaS
Logs: firewall, WAF, VNet flow logs, DNS → Log Analytics → Sentinel
```

Example Virtual Network Manager security admin rule intent: deny inbound TCP 22/3389 from `Internet` to all VNets in the organisation, which NSGs can't override.

Example firewall application rule collection (egress allow-list for an app spoke):

```text
Source: 10.20.0.0/16 (app spoke)
Allow HTTPS: pypi.org, files.pythonhosted.org, api.partner.example, *.blob.core.windows.net (only if not using PEs)
Allow FQDN tag: WindowsUpdate (for VM subnets)
Default: deny + log
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Hub-and-spoke vs Virtual WAN** | Hub-and-spoke for control and customisation; Virtual WAN for large, global, branch-heavy estates with routing intent. |
| **Firewall SKU** | Premium where TLS inspection/IDPS is required (regulated, internet-facing); Standard otherwise; Basic for small estates. |
| **TLS inspection** | Valuable but complex (certificates, privacy, breaking pinned apps); apply selectively. |
| **NSG vs firewall** | NSGs for micro-segmentation within VNets; firewall for cross-boundary inspection and FQDN egress. |
| **Network security perimeter vs private endpoints** | Private endpoints for client-to-PaaS; network security perimeter for PaaS-to-PaaS logical boundaries and controlling public access. |
| **Cost** | Firewalls, DDoS Network Protection and private endpoints add cost; share hubs and centralise where sensible. |

## Hands-on

1. Draw your current network with trust zones; mark every path that crosses a zone without inspection.
2. Deploy a hub with Azure Firewall and route a spoke's internet egress through it with an FQDN allow-list.
3. Replace a public storage endpoint with a private endpoint; disable public network access; verify DNS.
4. Create an AVNM security admin rule denying RDP/SSH from the internet.
5. Enable VNet flow logs and firewall logs to Sentinel; write a query for denied egress attempts.

## Security, cost and reliability notes

- **DNS is critical**: private endpoints fail without correct private DNS; centralise zones and resolver.
- **Asymmetric routing** with firewalls breaks traffic; design UDRs carefully.
- **Firewall as a single point**: deploy zone-redundant; plan capacity.
- **Exceptions creep**: track every public endpoint and egress exception with an owner.
- **Logging cost**: flow logs and firewall logs are high-volume; use the Sentinel data lake tier for retention (file 06).

## Common mistakes

| Mistake | Fix |
|---|---|
| Flat VNets with allow-all NSGs | Segment and deny by default |
| Private endpoints with public access still enabled | Disable public network access |
| Unrestricted internet egress | Firewall with allow-lists |
| Public RDP/SSH "temporarily" | Bastion and JIT; AVNM deny rules |
| No network logs in the SIEM | Flow, firewall, WAF, DNS logs to Sentinel |

## Check yourself

1. List the network security design principles.
2. Where should inbound, east-west and egress traffic be inspected?
3. What changed for outbound access on 31 March 2026?
4. How do AVNM security admin rules differ from NSGs?
5. When would you choose Firewall Premium?
6. How should users reach private applications without broad VPN access?

<details>
<summary>Answers</summary>

1. Segment, inspect, private by default, control egress, protect ingress, secure administration, monitor.
2. Inbound HTTP at WAF (Front Door/App Gateway); east-west and egress at Azure Firewall in the hub; hybrid at the firewall between on-premises and Azure.
3. Default outbound access was retired for new VNets; new subnets are private by default and need an explicit outbound method.
4. They're centrally managed, organisation-wide rules evaluated before NSGs and can't be overridden by workload teams.
5. When you need TLS inspection, IDPS, URL filtering or web categories, typically for regulated or internet-facing workloads.
6. Microsoft Entra Private Access (Global Secure Access) with per-app, identity-based access.

</details>

## References

- [Well-Architected: networking and connectivity security](https://learn.microsoft.com/azure/well-architected/security/networking)
- [Azure network security best practices](https://learn.microsoft.com/azure/security/fundamentals/network-best-practices)
- [Hub-spoke network topology](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke)
- [Azure Firewall overview](https://learn.microsoft.com/azure/firewall/overview)
- [Network security perimeter](https://learn.microsoft.com/azure/private-link/network-security-perimeter-concepts)
- [Default outbound access](https://learn.microsoft.com/azure/virtual-network/ip-services/default-outbound-access)
- [Global Secure Access](https://learn.microsoft.com/entra/global-secure-access/overview-what-is-global-secure-access)
