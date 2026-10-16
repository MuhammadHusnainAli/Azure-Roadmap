# Load Balancing Decision Guide

> Topic: 05 · Networking · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure has four load-balancing services plus several that overlap with them, and choosing between them is one of the most common architecture and exam questions. The choice follows from two properties of the traffic: is it HTTP or not, and is the scope one region or many. This file gives you the decision in a form you can apply in seconds, then the details that refine it.

## Key concepts

### The two questions

```text
                         Global (multi-region)          Regional (one region)
                    ┌──────────────────────────────┬──────────────────────────────┐
 HTTP(S)            │  Azure Front Door            │  Application Gateway         │
 (layer 7)          │                              │  (or App Gateway for         │
                    │                              │   Containers)                │
                    ├──────────────────────────────┼──────────────────────────────┤
 Non-HTTP           │  Traffic Manager (DNS)       │  Azure Load Balancer         │
 (layer 4 / DNS)    │  Cross-region Load Balancer  │                              │
                    └──────────────────────────────┴──────────────────────────────┘
```

### The four services

| | **Load Balancer** | **Application Gateway** | **Front Door** | **Traffic Manager** |
|---|---|---|---|---|
| OSI layer | 4 (TCP/UDP) | 7 (HTTP/S) | 7 (HTTP/S) | DNS |
| Scope | Regional (a global tier exists) | Regional | Global | Global |
| In the data path | Yes (pass-through) | Yes (reverse proxy) | Yes (reverse proxy at the edge) | No. It answers DNS only |
| Public / internal | Both | Both | Public entry (private origins supported) | Public |
| TLS termination | No | Yes | Yes | No |
| Web Application Firewall | No | Yes | Yes | No |
| Path / host routing | No | Yes | Yes | No |
| Caching / CDN | No | No | Yes | No |
| Session affinity | Source IP hash | Cookie | Cookie | No |
| Health probes | TCP, HTTP, HTTPS | HTTP, HTTPS | HTTP, HTTPS | HTTP, HTTPS, TCP |
| Failover speed | Seconds | Seconds | Seconds | Depends on DNS TTL and client caching |
| Typical use | Internal tiers, non-HTTP services, NVAs | Web apps and APIs in one region | Multi-region web apps, edge acceleration and security | Global routing for any protocol; hybrid and non-Azure endpoints |

### Azure Load Balancer

A layer 4, pass-through load balancer. It does not terminate connections; backend instances see the client's IP.

| Feature | Detail |
|---|---|
| **SKU** | **Standard** (the only current choice; Basic was retired on 30 September 2025). Zone-redundant or zonal, secure by default (closed until an NSG allows), SLA-backed |
| **Frontend** | Public IP, or private IP for an internal load balancer |
| **Rules** | Map frontend IP and port to a backend pool and port |
| **HA ports** | One rule that load-balances **all** ports and protocols on an internal load balancer. Used in front of NVAs |
| **Outbound rules** | Explicit SNAT configuration for outbound connectivity (NAT Gateway is usually better) |
| **Inbound NAT rules** | Port-forward to specific instances |
| **Global tier (cross-region)** | A static anycast IP in front of regional Standard load balancers, for global layer 4 failover |
| **Gateway Load Balancer** | Transparently inserts NVAs (firewalls, inspection appliances) into a traffic path |
| **Distribution** | Five-tuple hash by default; optional source-IP affinity |

Very fast, cheap, and unaware of HTTP.

### Application Gateway

A regional layer 7 reverse proxy with an optional Web Application Firewall. Covered in file 08. Choose it for HTTP workloads in a single region when you need TLS termination, path or host routing, WAF, or a private-only layer 7 entry point.

**Application Gateway for Containers** is the Kubernetes-native successor to the Application Gateway Ingress Controller, driven by Gateway API or Ingress resources in AKS.

### Azure Front Door

A global layer 7 entry point on Microsoft's edge network: anycast, TLS termination at the edge, WAF, caching, rules, and health-based routing across regions. Covered in file 09. Choose it when users or origins span regions, or when you want edge security and acceleration even for one region.

### Traffic Manager

A DNS-based router. Clients ask DNS for your name; Traffic Manager returns the address of the best endpoint according to the profile's routing method, and the client connects **directly** to that endpoint.

| Routing method | Returns |
|---|---|
| **Priority** | The highest-priority healthy endpoint (active-passive failover) |
| **Weighted** | Endpoints in proportion to weights (gradual migration, A/B) |
| **Performance** | The endpoint with the lowest latency for the client's network |
| **Geographic** | An endpoint by the client's location (data sovereignty, regional content) |
| **Multivalue** | Several healthy endpoints at once |
| **Subnet** | An endpoint by client IP range |

Characteristics:

- Works for **any protocol**, since it only handles DNS.
- Endpoints can be Azure, on-premises or other clouds.
- Failover is bounded by **DNS TTL** and client or resolver caching, so it is slower and less predictable than Front Door.
- It adds no security, no TLS termination and no inspection.
- Profiles can be **nested** to combine methods.

Use it for non-HTTP global routing, for hybrid and multicloud endpoints, or as a DNS layer above other services (for example failing over between two Front Door-fronted stacks or to a different CDN).

### Related services that also balance load

| Service | Role |
|---|---|
| **API Management** | API gateway. Load-balances and circuit-breaks across backend pools; the standard way to spread calls across several model deployments or regions (topic 11) |
| **Container Apps / App Service built-in ingress** | Each has its own HTTP ingress that balances across replicas |
| **AKS ingress and service mesh** | In-cluster layer 7 routing |
| **Azure Firewall** | Not a load balancer, but DNAT rules publish services |
| **NAT Gateway** | Outbound only (file 11) |

## How it works on Azure

### Decision flow

```text
1. Is the traffic HTTP or HTTPS?
   ├─ No  → 2
   └─ Yes → 3

2. Non-HTTP
   ├─ One region                         → Azure Load Balancer (public or internal)
   ├─ Multiple regions, need fast failover on one IP → Cross-region (global) Load Balancer
   └─ Multiple regions / hybrid, DNS-level routing   → Traffic Manager (in front of regional load balancers)

3. HTTP(S)
   ├─ Internet-facing, multi-region, or want edge WAF / CDN / acceleration → Azure Front Door
   │      └─ need regional L7 features too (private routing, mTLS to backends) → Front Door + Application Gateway
   ├─ Single region, internet-facing or internal                            → Application Gateway (WAF v2)
   ├─ AKS workloads wanting Kubernetes-native config                        → Application Gateway for Containers
   ├─ Internal API traffic needing policies, quotas, auth                   → API Management
   └─ Simple app on Container Apps / App Service with no WAF requirement    → the platform's built-in ingress
```

### Common combinations

| Pattern | Layout | Why |
|---|---|---|
| **Global web app** | Front Door → regional App Service or Container Apps (origin locked to Front Door, ideally through Private Link) | Edge WAF and caching, automatic regional failover |
| **Global with regional gateways** | Front Door → Application Gateway (per region) → backends | Regional path routing or private backends; avoid enabling WAF at both layers without a reason |
| **Internal line-of-business app** | Internal Application Gateway → app | Private layer 7 entry with WAF |
| **Three-tier VM app** | Application Gateway → web tier → internal Load Balancer → app tier → database | Layer 7 at the edge, layer 4 between tiers |
| **Non-HTTP global service** | Traffic Manager → regional public Load Balancers | Protocol-agnostic global routing |
| **NVA high availability** | Internal Load Balancer with HA ports → firewall instances | Symmetric, health-probed flows |
| **AI API platform** | Front Door or Application Gateway (WAF) → API Management → multiple Foundry deployments | Edge protection, then token limits, auth and model load balancing |

### Load balancing for AI workloads

Two distinct layers, which are often confused:

| Layer | Balances | Service |
|---|---|---|
| **User traffic to your application** | Requests across app instances and regions | Front Door, Application Gateway, platform ingress |
| **Your application's calls to models** | Requests across model deployments, regions and capacity types | API Management (backend pools with priority, weight and circuit breaker), or client-side logic |

A model endpoint is not an ordinary backend: capacity is measured in tokens per minute, throttling returns 429 with `Retry-After`, and you often want to prefer provisioned capacity and spill over to pay-as-you-go. Generic load balancers cannot do that. API Management's AI gateway features can.

Things to check when a layer 7 proxy sits in front of an AI application:

- **Streaming responses** (server-sent events): disable response buffering and confirm the proxy forwards chunks as they arrive.
- **Timeouts:** long generations can exceed default idle or response timeouts. Raise them deliberately.
- **WebSockets and real-time audio:** confirm protocol support on every hop.
- **Request size limits:** large prompts, documents and images can exceed WAF body inspection limits.
- **WAF false positives:** prompts and code in request bodies can match injection rules. Tune with exclusions, not by switching the WAF off.

### Health probes

A load balancer is only as good as its probe.

- Probe a **dedicated health endpoint** that checks the instance can do useful work, not just that the process is up.
- Keep it **cheap and fast**; it is called constantly.
- Do **not** call paid downstream services (such as a model) on every probe. Check a cached or lightweight dependency status instead.
- Allow the probe source in NSGs (`AzureLoadBalancer` tag for Load Balancer and Application Gateway health; `AzureFrontDoor.Backend` for Front Door).
- Understand the failure mode when **all** backends are unhealthy; some services then send traffic to all of them.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Front Door for a single-region app | Reasonable for edge WAF, TLS and caching, and it makes adding a region later painless. Otherwise Application Gateway is sufficient |
| Front Door and Application Gateway together | Only when you need capabilities from both. Each added hop adds cost, latency and a place for timeouts to mismatch |
| Traffic Manager or Front Door for HTTP | Front Door: faster failover, in-path health, WAF. Traffic Manager for non-HTTP or as a DNS layer above |
| Public or private entry | Internal Application Gateway or Private Link-backed Front Door origins for anything that should not be directly exposed |
| Where TLS terminates | At the edge for inspection and offload, then re-encrypt to the backend. End-to-end TLS is the default expectation |
| Platform ingress only | Fine for internal or low-risk apps. Add WAF for internet-facing production |

**Cost shape.** Load Balancer is the cheapest; Application Gateway and Front Door Premium have meaningful fixed monthly costs plus capacity or request charges; Traffic Manager is priced per DNS query and endpoint check. For small workloads the entry point can cost more than the application.

## Hands-on

No deployment. Choose the service, or combination, for each scenario and justify it in one sentence.

1. A public web shop running in West Europe and East US, needing WAF and static content caching.
2. An internal HR portal in one region, reachable only from the corporate network, with OWASP protection.
3. A fleet of game servers using UDP in three regions.
4. Two firewall NVAs in a hub that must both receive traffic symmetrically.
5. A SQL Server Always On listener inside a VNet.
6. An SFTP service that must fail over between an Azure region and an on-premises data centre.
7. A public chat application on Container Apps in one region, streaming responses, with a WAF requirement.
8. An application that calls GPT models in three regions, preferring provisioned capacity and spilling to pay-as-you-go on 429.
9. Microservices on AKS that the platform team wants to expose using Gateway API resources.
10. A migration that should shift 10% of users to a new stack this week and 50% next week, across two unrelated hosting platforms.

<details>
<summary>Suggested answers</summary>

1. Azure Front Door (Premium for managed WAF rules and Private Link origins): global HTTP with edge WAF and caching.
2. Internal Application Gateway with WAF v2: regional, private, layer 7.
3. Traffic Manager with performance routing in front of regional public Load Balancers: non-HTTP, global. (Or a cross-region Load Balancer.)
4. Internal Standard Load Balancer with HA ports: all ports, symmetric flows, health-probed.
5. Internal Standard Load Balancer: layer 4 inside the VNet.
6. Traffic Manager with priority routing: protocol-agnostic and supports external endpoints.
7. Application Gateway WAF v2 (or Front Door) in front of the container app, with buffering disabled and timeouts raised for streaming.
8. API Management with a backend pool using priorities and circuit breakers: model-aware load balancing.
9. Application Gateway for Containers.
10. Traffic Manager with weighted routing (or Front Door weighted origins if both are HTTP and reachable as origins).

</details>

## Security, cost and reliability notes

- **Security:** restrict backends so they accept traffic **only** from the load balancer: NSG rules, access restrictions, the `X-Azure-FDID` header check for Front Door, or Private Link origins. Otherwise attackers bypass the WAF by calling the backend directly.
- **Security:** Load Balancer and Traffic Manager provide no application-layer protection.
- **Cost:** stacked layers multiply both fixed charges and per-GB processing.
- **Reliability:** deploy zone-redundant SKUs. A zonal entry point in front of a zone-redundant application is a single point of failure.
- **Reliability:** align timeouts across every hop. The shortest one wins, and mismatches cause intermittent 502 and 504 errors.

## Common mistakes

- Using Application Gateway alone for a multi-region application.
- Using Traffic Manager for HTTP where Front Door's faster, in-path failover is needed.
- Expecting Load Balancer to do TLS termination or URL routing.
- Stacking Front Door, Application Gateway and API Management without needing all three.
- WAF enabled at two layers with duplicate rule sets and double the false positives.
- Backends reachable directly, bypassing the WAF.
- Health probes that hit the home page, or that call a paid API.
- Default timeouts cutting off streamed model responses.
- Trying to balance across model deployments with a generic load balancer.
- Still planning around Basic SKUs.

## Check yourself

1. Which two questions determine the choice of load balancer?
2. Which services operate at layer 7, and which is global?
3. How does Traffic Manager differ fundamentally from the other three?
4. When would you use an internal Load Balancer with HA ports?
5. Why is Front Door usually preferred over Traffic Manager for web applications?
6. What balances an application's calls across multiple model deployments, and why not a generic load balancer?
7. What must you configure on a proxy in front of a streaming chat application?
8. Why must backends be locked down to the load balancer?

<details>
<summary>Answers</summary>

1. Is the traffic HTTP(S) or not, and is the scope global or regional.
2. Application Gateway (regional) and Front Door (global).
3. It works only at the DNS level and is not in the data path; clients connect directly to the endpoint it returns.
4. In front of network virtual appliances, to load-balance all ports and protocols with symmetric flows.
5. It sits in the data path, so it fails over in seconds without depending on DNS TTLs, and adds WAF, TLS termination and caching.
6. API Management with backend pools, priorities and circuit breakers, because model capacity is token-based and throttling needs 429-aware handling and spillover logic.
7. Disable response buffering, raise idle and response timeouts, and check body size limits and WAF exclusions.
8. Otherwise clients can reach the backend directly and bypass the WAF and routing controls.

</details>

## References

- [Load-balancing options](https://learn.microsoft.com/azure/architecture/guide/technology-choices/load-balancing-overview)
- [What is Azure Load Balancer?](https://learn.microsoft.com/azure/load-balancer/load-balancer-overview)
- [Cross-region (global) load balancer](https://learn.microsoft.com/azure/load-balancer/cross-region-overview)
- [Gateway Load Balancer](https://learn.microsoft.com/azure/load-balancer/gateway-overview)
- [What is Traffic Manager?](https://learn.microsoft.com/azure/traffic-manager/traffic-manager-overview)
- [Traffic Manager routing methods](https://learn.microsoft.com/azure/traffic-manager/traffic-manager-routing-methods)
- [What is Application Gateway for Containers?](https://learn.microsoft.com/azure/application-gateway/for-containers/overview)
- [Health Endpoint Monitoring pattern](https://learn.microsoft.com/azure/architecture/patterns/health-endpoint-monitoring)
- [AI gateway capabilities in Azure API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
