# 05 · Networking

> Tracks: E ○ A ● C ○ · Part B · Core Infrastructure and Application Platform

Networking decides who can reach what. In Azure almost every production design, and every serious AI design, comes down to the same handful of questions: where is the private boundary, how do names resolve inside it, how does traffic get in, how does it get out, and how do you prove it. This topic builds those answers piece by piece and finishes by applying them to AI workloads.

## What you will be able to do

- Plan address space and subnets for a workload and a landing zone.
- Control traffic with NSGs, routes and firewalls, and explain how they are evaluated.
- Make PaaS services private with Private Link and get DNS right.
- Choose correctly between Load Balancer, Application Gateway, Front Door and Traffic Manager.
- Design ingress with a WAF and egress through NAT or a firewall.
- Connect on-premises networks with VPN and ExpressRoute.
- Troubleshoot "it can't connect" methodically.
- Design a network-isolated Foundry workload end to end.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) and [Topic 03](../03-governance-and-landing-zones/README.md) (landing zones). Basic IP networking: CIDR notation, TCP and UDP ports, DNS record types, what TLS does.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Virtual Networks and Subnets](01-virtual-networks-and-subnets.md) | 50 min | CLI |
| 02 | [Network Security Groups and Application Security Groups](02-network-security-groups-and-application-security-groups.md) | 45 min | CLI |
| 03 | [Routing](03-routing.md) | 40 min | CLI |
| 04 | [VNet Peering and Azure Virtual Network Manager](04-vnet-peering-and-azure-virtual-network-manager.md) | 45 min | CLI |
| 05 | [Private Link and Private Endpoints](05-private-link-and-private-endpoints.md) | 60 min | CLI |
| 06 | [Azure DNS](06-azure-dns.md) | 50 min | CLI |
| 07 | [Load Balancing Decision Guide](07-load-balancing-decision-guide.md) | 35 min | – |
| 08 | [Application Gateway and Web Application Firewall](08-application-gateway-and-web-application-firewall.md) | 50 min | CLI |
| 09 | [Azure Front Door and CDN](09-azure-front-door-and-cdn.md) | 45 min | CLI |
| 10 | [Azure Firewall and Firewall Manager](10-azure-firewall-and-firewall-manager.md) | 50 min | Review |
| 11 | [NAT Gateway and Outbound Connectivity](11-nat-gateway-and-outbound-connectivity.md) | 35 min | CLI |
| 12 | [VPN Gateway and ExpressRoute](12-vpn-gateway-and-expressroute.md) | 50 min | Review |
| 13 | [Virtual WAN](13-virtual-wan.md) | 30 min | – |
| 14 | [Azure Bastion and Secure Admin Access](14-azure-bastion-and-secure-admin-access.md) | 30 min | Portal |
| 15 | [DDoS Protection](15-ddos-protection.md) | 25 min | – |
| 16 | [Network Security Perimeter](16-network-security-perimeter.md) | 35 min | CLI |
| 17 | [Network Watcher and Troubleshooting](17-network-watcher-and-troubleshooting.md) | 45 min | CLI |
| 18 | [Networking for AI Workloads](18-networking-for-ai-workloads.md) | 75 min | Design |

**AI engineers on a tight schedule:** read 01, 02, 05, 06, 11, 17 and 18. Those cover what blocks AI deployments in practice.

## Completion checklist

- [ ] I can carve a /16 into subnets for a hub or spoke and justify each size.
- [ ] I can state the NSG default rules and the order of evaluation.
- [ ] I can predict the next hop for a packet given system routes, UDRs and BGP routes.
- [ ] I have created a private endpoint and verified the name resolves to a private IP.
- [ ] I can draw the DNS resolution path from on-premises to a private endpoint.
- [ ] I can choose a load balancer for any scenario in one sentence.
- [ ] I can explain why a new VM in a new VNet has no internet access by default.
- [ ] I can list the subnets, private endpoints and DNS zones a private Foundry workload needs.
- [ ] I have used Network Watcher to diagnose a blocked connection.

## Maps to

- **AZ-104**: implement and manage virtual networking.
- **AZ-700**: the whole topic, in more depth.
- **AZ-305**: design network solutions.
- **AI-103**: secure AI solutions with private networking.
- **SC-500**: network security controls.

## Next

[06 · Compute](../06-compute/README.md)
