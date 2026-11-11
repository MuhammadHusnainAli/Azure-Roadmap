# Zero Trust Architecture

## Why it matters

Most modern breaches don't start by breaking through a firewall. They start with a phished password, a stolen token, an over-permissioned service principal or a leaked key, then move laterally to whatever that identity can reach. A security model that trusts anything "inside the network" can't stop that. **Zero Trust** replaces implicit trust with continuous verification: every request is authenticated and authorised using all available signals, access is least privilege and time-bound, and controls assume an attacker is already inside. It's the organising principle for everything else in this topic, and the language security architects, auditors and executives use to discuss cloud security strategy.

## Key concepts

### Three principles

| Principle | Meaning | Azure examples |
|---|---|---|
| **Verify explicitly** | Authenticate and authorise every request using identity, device, location, behaviour and risk | Entra ID with MFA, Conditional Access, device compliance, ID Protection risk |
| **Use least privilege access** | Just enough access, for the shortest time | Azure RBAC at narrow scope, PIM just-in-time roles, managed identities, access reviews |
| **Assume breach** | Limit blast radius; detect and respond fast | Segmentation, private endpoints, encryption, Defender for Cloud, Sentinel, immutable backups |

Microsoft's framing of outcomes: access is explicitly granted and continuously evaluated; access is conditional and temporary; permissions are tightly scoped; controls apply consistently across on-premises, cloud, SaaS and AI; detection and response are built in. Security isn't only technology: people and processes create risk, and everyone shares responsibility.

### Pillars

| Pillar | Goal | Key Microsoft controls |
|---|---|---|
| **Identities** (users, workloads, agents) | Strong authentication, risk-based access | Entra ID, phishing-resistant MFA, Conditional Access, PIM, workload and agent identities ([Topic 02](../02-identity-and-access/README.md)) |
| **Endpoints** | Only healthy, managed devices access resources | Intune, Defender for Endpoint, device compliance in Conditional Access |
| **Applications** | Apps authenticate users and each other properly; shadow IT discovered | App registrations, Defender for Cloud Apps, API Management |
| **Data** | Classify, label, protect and monitor data wherever it goes | Purview Information Protection, DLP, encryption, Key Vault (files 03, 04, 10) |
| **Infrastructure** | Hardened, monitored, policy-governed resources | Azure Policy, Defender for Cloud, Microsoft Cloud Security Benchmark (files 02, 05) |
| **Network** | Segment, inspect, prefer private access, control egress | VNets, NSGs, Azure Firewall, Private Link, WAF, DDoS (file 07) |
| **Visibility, automation, orchestration** | Correlate signals and respond automatically | Defender XDR, Sentinel, Logic Apps playbooks (files 06, 12) |

### Network perimeter vs Zero Trust

| Traditional | Zero Trust |
|---|---|
| Trust based on network location | Trust based on verified identity, device and context |
| Flat internal networks | Micro-segmentation; private endpoints per service |
| VPN gives broad access | Per-application access (Entra Private Access, Bastion) |
| Security team owns security | Shared responsibility across teams |
| Periodic checks | Continuous evaluation (Continuous Access Evaluation) |

### Maturity

Zero Trust is a journey. Typical stages: **traditional** (static rules, on-premises identity), **advanced** (cloud identity, Conditional Access, some segmentation, analytics), **optimal** (risk-adaptive access, just-in-time everything, automated response, full telemetry). The Microsoft Zero Trust guidance and workshop help assess each pillar.

## How it works on Azure

Applying Zero Trust to an Azure workload (from Microsoft's Azure infrastructure guidance):

| Layer | Controls |
|---|---|
| **Management plane** | Entra ID only; Conditional Access requiring phishing-resistant MFA and compliant devices for admins; PIM for Owner/Contributor; no standing privileged access; break-glass accounts monitored |
| **Workload identities** | Managed identities; no secrets in code; Key Vault for unavoidable secrets; workload identity federation for CI/CD |
| **Network** | Hub-and-spoke or Virtual WAN; Azure Firewall for east-west and egress; NSGs per subnet; Private Link for PaaS; public network access disabled; Bastion for admin access |
| **Data** | Encryption at rest (CMK where required), TLS 1.2+, sensitivity labels, Purview |
| **Threat protection** | Defender for Cloud plans; Defender XDR; Sentinel analytics |
| **Governance** | Azure Policy initiatives (Microsoft Cloud Security Benchmark), landing zone guardrails, regulatory compliance dashboard |

Example Conditional Access intent for Azure management (implemented in Entra):

```text
Users: All users with Azure RBAC privileged roles (via group)
Target resource: Windows Azure Service Management API
Conditions: any location, any device platform
Grant: require authentication strength "Phishing-resistant MFA" AND require compliant device
Session: sign-in frequency 4 hours
```

And a policy to deny public network access on storage (Azure Policy built-in "Storage accounts should disable public network access") assigned at the landing zones management group.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Start with identity** | Highest return: MFA, Conditional Access, PIM, managed identities. |
| **Private by default** | Private endpoints for data services; exceptions documented (for example public front ends behind WAF). |
| **Segmentation granularity** | Per workload and tier; balance with operational complexity. |
| **Continuous verification** | Enable Continuous Access Evaluation and risk-based policies. |
| **Legacy systems** | Isolate and wrap with modern access (application proxy, Private Access) where they can't support modern auth. |
| **AI and agents** | Same principles: per-agent identity, least agency, monitored tool access ([Topic 23 file 04](../23-responsible-ai-security-and-governance/04-securing-agents.md)). |

## Hands-on

1. Assess one workload against the seven pillars; score each traditional/advanced/optimal.
2. Create Conditional Access requiring phishing-resistant MFA for Azure management for a pilot admin group.
3. Configure PIM for the Contributor role on a subscription with approval and justification.
4. Disable public network access on a storage account and connect via private endpoint.
5. List all identities with Owner rights across subscriptions using Resource Graph and plan reduction.

## Security, cost and reliability notes

- **Lockout risk**: test Conditional Access in report-only mode; keep monitored break-glass accounts excluded.
- **Cost**: Entra ID P1/P2, Defender plans and firewalls cost money; prioritise by risk.
- **Operational friction**: just-in-time access adds steps; automate approvals for low-risk roles.
- **Reliability**: private networking adds DNS and routing dependencies; design and test them ([Topic 05](../05-networking/README.md)).

## Common mistakes

| Mistake | Fix |
|---|---|
| Equating Zero Trust with a product | It's a strategy across pillars |
| Network-only focus | Identity first |
| Standing Owner access for admins | PIM and just-in-time |
| Secrets in app settings | Managed identities and Key Vault |
| One-time project | Continuous improvement with metrics |

## Check yourself

1. State the three Zero Trust principles.
2. Name the Zero Trust pillars.
3. How does Zero Trust differ from perimeter security?
4. Give three Azure controls each for identity and network.
5. Why start with identity?
6. What are the risks when rolling out Conditional Access, and how do you mitigate them?

<details>
<summary>Answers</summary>

1. Verify explicitly, use least privilege access, assume breach.
2. Identities, endpoints, applications, data, infrastructure, network, plus visibility/automation/orchestration.
3. Trust is based on verified identity, device and context rather than network location, with continuous evaluation and segmentation.
4. Identity: MFA, Conditional Access, PIM, managed identities. Network: NSGs, Azure Firewall, Private Link, WAF, Bastion.
5. Most attacks exploit identities; identity controls protect all resources regardless of network location.
6. Locking out admins or users; use report-only mode, pilot groups, and excluded monitored break-glass accounts.

</details>

## References

- [Zero Trust as a security foundation](https://learn.microsoft.com/security/zero-trust/zero-trust-overview)
- [Apply Zero Trust to Azure infrastructure](https://learn.microsoft.com/security/zero-trust/azure-infrastructure-overview)
- [Microsoft cloud security benchmark](https://learn.microsoft.com/security/benchmark/azure/overview)
- [Network security best practices](https://learn.microsoft.com/azure/security/fundamentals/network-best-practices)
