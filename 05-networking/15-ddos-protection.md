# DDoS Protection

## Why it matters

A distributed denial-of-service attack tries to make a service unavailable by overwhelming it. Any internet-facing endpoint is a potential target, and attacks are cheap to launch. Azure absorbs a great deal automatically, but the default protection is tuned to protect the platform, not your particular application. Architects need to know what is included, what the paid tiers add, and why AI endpoints have an extra kind of exposure that DDoS products do not cover.

## Key concepts

### Attack types

| Type | Layer | Example | Mitigated by |
|---|---|---|---|
| **Volumetric** | 3/4 | UDP floods, amplification (DNS, NTP), ICMP floods | Azure DDoS Protection, Front Door's edge |
| **Protocol** | 3/4 | SYN floods, fragmented packets, reflection | Azure DDoS Protection |
| **Resource / application layer** | 7 | HTTP floods, slow-rate attacks, expensive-query abuse | **Web Application Firewall** (rate limiting, bot protection), application design |

DDoS Protection handles the first two. **Layer 7 attacks need a WAF.** A complete defence uses both.

### Tiers

| | **Infrastructure protection** | **IP Protection** | **Network Protection** |
|---|---|---|---|
| Cost | Free, always on | Per protected public IP, monthly | Fixed monthly plan fee covering up to 100 public IPs, then per IP |
| Scope | Every Azure public endpoint | Individual public IPs you enable | All public IPs in the virtual networks linked to the plan |
| Thresholds | Tuned for Azure infrastructure as a whole | **Adaptive**, per IP, learned from your traffic | **Adaptive**, per IP |
| Attack telemetry, metrics, alerts, logs | No | Yes | Yes |
| Mitigation reports and flow logs | No | Yes | Yes |
| **DDoS Rapid Response** team engagement | No | No | Yes |
| **Cost protection** (credits for scale-out during a documented attack) | No | No | Yes |
| **WAF discount** | No | No | Yes |
| Integration with Firewall Manager, Sentinel, Defender for Cloud | No | Yes | Yes |

**Infrastructure protection** is why a small attack will not take down the Azure region. Because its thresholds are set for the platform, an attack large enough to overwhelm *your* application may be too small to trigger it.

**IP Protection** suits a few public IPs. **Network Protection** suits organisations with many, and adds the support and financial guarantees. At roughly fifteen or more public IPs, or when rapid response and cost protection matter, Network Protection is the better buy.

### How protection works

1. **Always-on monitoring** of traffic to each protected public IP.
2. **Adaptive tuning:** the service learns the normal traffic profile for each IP and sets mitigation thresholds for TCP SYN, TCP and UDP accordingly.
3. **Detection:** when traffic exceeds the threshold, mitigation starts automatically within seconds.
4. **Scrubbing:** attack traffic is dropped at Azure's edge; legitimate traffic is forwarded.
5. **Telemetry:** metrics, alerts and reports show what happened.

There is nothing to deploy in the data path and no change to latency under normal conditions.

### What gets protected

Public IP addresses attached to resources in a protected virtual network: VMs, load balancers, Application Gateway, Azure Firewall, Bastion, VPN gateways, API Management in a VNet, and others.

Not covered by a virtual network plan, because they are not in your VNet:

| Service | Protection |
|---|---|
| **Azure Front Door, CDN** | Built-in DDoS protection at Microsoft's edge |
| **App Service, Functions, Container Apps public endpoints, Storage, other multi-tenant PaaS** | Platform infrastructure protection. Front them with Front Door or Application Gateway for more |
| **Public IP prefixes, BYOIP ranges** | Supported with conditions |

### Reducing the attack surface

The best mitigation is having little to attack:

- **Fewer public IPs.** Private endpoints, internal load balancers and private subnets mean fewer targets.
- **One hardened entry point.** Front Door or Application Gateway with WAF, with origins unreachable directly.
- **Scale-out capacity** and autoscale so the application absorbs spikes that get through.
- **No single points of failure** behind the entry: zone redundancy, multiple instances.

## How it works on Azure

### Layered defence for a web application

```text
Internet
   │
   ▼
Azure Front Door (edge DDoS absorption, WAF: rate limiting, bot protection, managed rules)
   │  Private Link
   ▼
Application / Application Gateway in a VNet with DDoS Network Protection (protects any remaining public IPs)
   │
   ▼
Autoscaling application with health-based load balancing
```

If Front Door is the only public entry and origins are private, the virtual network may have very few public IPs left to protect: perhaps the firewall's and gateway's. Whether to buy a paid plan then depends on how critical those are.

### In a landing zone

- Create **one DDoS Network Protection plan** in the connectivity subscription.
- **Link** every virtual network that has public IPs to it. The plan works across subscriptions in the same tenant.
- An Azure Policy (`modify` or `deployIfNotExists`) can enable the plan on VNets automatically.
- Send diagnostics for protected public IPs to the central Log Analytics workspace and Sentinel.
- Configure **alerts** on the *Under DDoS attack or not* metric.

CAF guidance for AI workloads: activate DDoS Protection for **internet-facing** AI workloads.

### The AI-specific problem: economic denial of service

DDoS products count packets and requests. An AI endpoint can be attacked far below those thresholds, because each request is **expensive**:

| Attack | Effect |
|---|---|
| A modest stream of long, complex prompts | Consumes tokens-per-minute quota, throttling real users (a denial of service) |
| Automated use of your chatbot as a free model API | Runs up the bill (a denial of wallet) |
| Requests engineered for maximum output length or tool loops | Amplifies cost per request |
| Many accounts or IPs, each under a naive rate limit | Evades simple controls |

None of this looks like a flood. Defences are at layer 7 and in the application:

| Control | Where |
|---|---|
| **Require authentication** before any model call | Application, Entra ID |
| **Rate limit per user or client**, not just per IP | WAF custom rules; API Management |
| **Token limits per consumer** (tokens per minute, monthly quotas) | API Management token-limit policies |
| **Cap `max_tokens`**, prompt size, conversation length, tool iterations | Application |
| **Bot protection and challenges** | WAF |
| **Budgets and alerts** on model spend | Cost Management |
| **Abuse monitoring and anomaly detection** on usage per user | Application telemetry |
| **Separate quota** for critical internal workloads from public-facing ones | Separate deployments or subscriptions |

Topics 11 and 23 cover these controls in depth.

### Responding to an attack

1. **Detect:** alert on the *Under DDoS attack* metric and on application health.
2. **Confirm:** check mitigation metrics (packets dropped and forwarded), WAF logs, application metrics.
3. **Engage:** with Network Protection, open a support case to reach DDoS Rapid Response during an active attack.
4. **Adapt:** add WAF rate-limit or geo rules; scale out; shed non-critical load.
5. **Review:** read the post-mitigation report; apply for cost protection if eligible; fix whatever weakness was exposed.

Have this written down and rehearsed. **Simulation testing** through Microsoft's approved partners (for example BreakingPoint Cloud, Red Button, RedWolf) lets you validate protection and your runbook against your own resources. Do not run unapproved attack tools against Azure.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Free tier only | Acceptable for internal workloads and non-critical public ones behind Front Door |
| IP Protection or Network Protection | IP Protection for a handful of critical IPs. Network Protection for many, or when rapid response and cost protection are wanted |
| One plan or several | One per tenant, shared across subscriptions |
| Front Door instead of a plan | Front Door protects what is behind it at the edge. You still need a plan for public IPs that do not sit behind it |
| WAF | Always, for internet-facing HTTP. DDoS Protection alone does not stop application-layer attacks |

**Cost.** Network Protection has a substantial fixed monthly fee, so it makes sense as a shared platform service, not per workload. IP Protection scales linearly per address.

## Hands-on

No attack simulation here. Explore configuration and reason about a design.

```bash
RG=rg-ddos-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# Inventory: how many public IPs do you have, what are they attached to, and are they protected?
az graph query -q "
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| project name, resourceGroup, ip = tostring(properties.ipAddress),
          attachedTo = tostring(properties.ipConfiguration.id),
          protectionMode = tostring(properties.ddosSettings.protectionMode)" -o table

# Which VNets have a DDoS plan enabled?
az graph query -q "
Resources
| where type =~ 'microsoft.network/virtualnetworks'
| project name, resourceGroup, ddosEnabled = tobool(properties.enableDdosProtection),
          plan = tostring(properties.ddosProtectionPlan.id)" -o table

# Enable IP Protection on a single public IP (billable per IP while enabled)
az network public-ip create -g $RG -n pip-ddos-lab --sku Standard -o none
az network public-ip update -g $RG -n pip-ddos-lab --protection-mode Enabled -o none
az network public-ip show -g $RG -n pip-ddos-lab --query ddosSettings

# Metrics available for a protected IP
az monitor metrics list-definitions \
  --resource $(az network public-ip show -g $RG -n pip-ddos-lab --query id -o tsv) \
  --query "[?contains(name.value,'DDoS') || contains(name.value,'Ddos')].name.value" -o tsv

az group delete -n $RG --yes --no-wait
```

Design exercises:

1. From the inventory, list each public IP and decide whether it could be removed (private endpoint, internal load balancer, Bastion instead of a VM public IP).
2. A public customer-support chatbot is fronted by Front Door Premium with private origins. The hub has a firewall and VPN gateway with public IPs. Recommend a DDoS tier and justify it.
3. For the same chatbot, list six controls against cost-driven abuse and say where each is implemented.

<details>
<summary>Notes on exercise 2</summary>

Front Door provides edge protection for the application. The remaining public IPs are the firewall and VPN gateway. If hybrid connectivity is business-critical and several workloads depend on the hub, Network Protection on the hub VNet is justified, shared across all workloads. If the hub is small and VPN is only a backup path, IP Protection on those two addresses, or the free tier with a documented risk acceptance, is defensible.

</details>

## Security, cost and reliability notes

- **Security:** DDoS attacks are sometimes a distraction for another intrusion. Keep watching other signals during one.
- **Security:** layer 3/4 protection plus a WAF plus application limits. No single layer is enough.
- **Cost:** Network Protection's cost protection covers scale-out and egress charges incurred during a documented attack. Without it, an attack can be expensive even if you stay up.
- **Reliability:** protection keeps the network path clear. The application still has to scale and degrade gracefully.
- **Reliability:** test. Untested protection and an unrehearsed runbook fail when they are needed.

## Common mistakes

- Assuming the free tier protects the application as well as it protects the platform.
- Buying DDoS Protection and skipping the WAF.
- Paying per workload instead of sharing one plan.
- Public IPs that never needed to exist.
- Origins reachable directly, so attackers bypass Front Door's protection.
- No alerting on attack metrics.
- Treating an AI endpoint's cost and quota exposure as a DDoS problem that the DDoS product will handle.
- Running unapproved load or attack tools against Azure endpoints.

## Check yourself

1. Which attack types does Azure DDoS Protection mitigate, and which need a WAF?
2. What does the free infrastructure tier not give you?
3. Compare IP Protection and Network Protection.
4. What is adaptive tuning?
5. How is an application behind Front Door protected, and what might still need a plan?
6. Why can an AI endpoint be denied service by traffic that no DDoS product would flag?
7. Name four controls against denial-of-wallet attacks on a model endpoint.

<details>
<summary>Answers</summary>

1. It mitigates volumetric and protocol attacks at layers 3 and 4. Application-layer (layer 7) attacks need a Web Application Firewall.
2. Thresholds tuned to your traffic, telemetry and alerts, mitigation reports, rapid response support and cost protection.
3. IP Protection is billed per public IP and offers adaptive tuning and telemetry. Network Protection is a plan covering all public IPs in linked VNets and adds DDoS Rapid Response, cost protection and a WAF discount.
4. The service learns each protected IP's normal traffic profile and sets mitigation thresholds accordingly.
5. Front Door absorbs and filters attacks at Microsoft's edge. Public IPs that are not behind it, such as firewalls and gateways in the VNet, may still need a plan.
6. Each request is costly in tokens and quota, so a small volume of expensive requests can exhaust capacity or budget without looking like a flood.
7. Any four of: authentication before model calls, per-user rate limits, token limits per consumer at the gateway, caps on output tokens and prompt size, bot protection, budgets and spend alerts, usage anomaly monitoring.

</details>

## References

- [What is Azure DDoS Protection?](https://learn.microsoft.com/azure/ddos-protection/ddos-protection-overview)
- [About Azure DDoS Protection tier comparison](https://learn.microsoft.com/azure/ddos-protection/ddos-protection-sku-comparison)
- [Azure DDoS Protection features](https://learn.microsoft.com/azure/ddos-protection/ddos-protection-features)
- [Azure DDoS Protection reference architectures](https://learn.microsoft.com/azure/ddos-protection/ddos-protection-reference-architectures)
- [Azure DDoS Rapid Response](https://learn.microsoft.com/azure/ddos-protection/ddos-rapid-response)
- [Test with simulation partners](https://learn.microsoft.com/azure/ddos-protection/test-through-simulations)
- [Application (Layer 7) DDoS protection](https://learn.microsoft.com/azure/web-application-firewall/shared/application-ddos-protection)
- [Limit large language model API token usage (API Management)](https://learn.microsoft.com/azure/api-management/llm-token-limit-policy)
