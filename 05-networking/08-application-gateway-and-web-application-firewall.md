# Application Gateway and Web Application Firewall

## Why it matters

Application Gateway is the regional layer 7 entry point for web applications and APIs in a virtual network, and its Web Application Firewall is the standard defence against common web attacks. It appears in nearly every reference architecture for network-isolated workloads, including the baseline Foundry chat architecture, because it is the piece that lets traffic from outside reach private application hosts safely.

## Key concepts

### What it is

A managed reverse proxy deployed **into a dedicated subnet** of your VNet. It terminates client connections, applies routing and security rules, and opens new connections to backends.

**SKUs**

| SKU | Notes |
|---|---|
| **Standard_v2** | Autoscaling, zone redundancy, static VIP, header rewrite, Key Vault integration |
| **WAF_v2** | Standard_v2 plus Web Application Firewall |
| **Basic** (v2 family) | Lower-cost option with reduced scale and features, for small workloads |
| v1 (Standard, WAF) | Retired in 2026. Migrate any that remain |

### Components

```text
Client ──► Frontend IP ──► Listener ──► Routing rule ──► Backend settings ──► Backend pool
            public /        port,          basic or          protocol, port,      VMs, VMSS,
            private         protocol,      path-based        timeout, probe,      App Service,
                            host name,                       affinity, host       IPs, FQDNs
                            certificate                      override
```

| Component | Role |
|---|---|
| **Frontend IP** | Public, private or both. A **private-only** deployment (no public IP) is supported |
| **Listener** | Port, protocol (HTTP, HTTPS, also TCP/TLS proxy), optional host name. **Multi-site** listeners host many sites on one gateway, including wildcards |
| **Routing rule** | Connects a listener to backend settings and a pool. **Basic** sends everything to one pool; **path-based** routes by URL path map. Rules have priorities |
| **Backend pool** | Targets by NIC, IP or FQDN: VMs, scale sets, App Service, Container Apps, internal load balancers, on-premises servers |
| **Backend settings** | Protocol and port to the backend, request timeout, cookie affinity, connection draining, host name override, custom probe, trusted root certificates |
| **Health probe** | Default or custom HTTP(S) probe per backend setting |
| **Rewrite rules** | Add, remove or change request and response headers and URL components, optionally with conditions |
| **Redirects** | HTTP to HTTPS, or to another listener or external site |
| **SSL profile / policy** | TLS versions and cipher suites; mutual TLS client authentication |

### TLS

| Mode | Description |
|---|---|
| **TLS termination** | Decrypt at the gateway, plain HTTP to the backend. Simplest; traffic inside the VNet is unencrypted |
| **End-to-end TLS** | Decrypt at the gateway (so the WAF can inspect), then re-encrypt to the backend. The recommended default |
| **Mutual TLS** | The gateway validates a client certificate; optionally passes it to the backend |
| **TLS/TCP proxy** | Layer 4 proxying for non-HTTP protocols |

Certificates are best stored in **Key Vault** and referenced through the gateway's managed identity, so renewals are picked up automatically. Use a predefined modern **TLS policy** (TLS 1.2 minimum, 1.3 supported) rather than the legacy default.

For backends, the gateway must trust the backend's certificate chain and the host name it presents must match. With App Service and Container Apps, the cleanest approach is to use the **same custom domain** on gateway and backend, avoiding host header overrides that break redirects and cookies.

### Scaling and availability

- **Autoscaling** between a minimum and maximum instance count, measured in capacity units. Set a minimum above zero for production to avoid scale-up delay.
- **Zone redundancy** across availability zones.
- A **static** public VIP.
- **Connection draining** for graceful backend removal.

### Subnet and networking requirements

- A **dedicated subnet**; `/24` is recommended for v2 to allow scale-out and upgrades.
- **NSG rules** required: inbound from `GatewayManager` on TCP 65200–65535 (v2 infrastructure), from `AzureLoadBalancer`, and from clients on your listener ports. Do not block outbound internet for the control plane unless using a private deployment configuration that supports it.
- **UDR restrictions:** a `0.0.0.0/0` route to a virtual appliance is constrained on the gateway subnet in public deployments. Private deployments relax several of these limits. Check the infrastructure configuration guidance before applying a blanket route table.
- Backends can be in the same VNet, peered VNets, or on-premises.

### Web Application Firewall

The WAF inspects HTTP requests and blocks malicious ones before they reach your application.

**Managed rule sets**

| Rule set | Covers |
|---|---|
| **Default Rule Set (DRS)**, Microsoft-managed, built on the OWASP Core Rule Set | SQL injection, cross-site scripting, local and remote file inclusion, command injection, protocol violations, and Microsoft Threat Intelligence rules |
| **OWASP CRS** (older versions) | Legacy option; prefer the current DRS |
| **Bot Manager** | Known bad bots blocked, good bots allowed, unknown bots handled per policy |
| **HTTP DDoS / rate-related protections** | Layer 7 flood mitigation features |

Current DRS versions use **anomaly scoring**: each matched rule adds to a score and the request is blocked when the total crosses a threshold, which reduces false positives compared with blocking on any single match.

**Custom rules**

Evaluated before managed rules, in priority order. Match on IP address, geography, request URI, headers, cookies, query strings, body, or method; act with allow, block, log, or JavaScript challenge. **Rate-limit** rules throttle by client IP or other keys.

Typical custom rules: allow only certain countries, block a known-abusive IP range, rate-limit a login or chat endpoint, require a header.

**Modes**

| Mode | Behaviour |
|---|---|
| **Detection** | Logs matches, blocks nothing. For tuning |
| **Prevention** | Blocks according to rules |

**Policy scope**

A **WAF policy** is a separate resource associated at three levels, most specific wins:

| Level | Use |
|---|---|
| Global (gateway) | Baseline for everything |
| Per-site (listener) | Different rules for different applications |
| Per-URI (path) | Different rules for specific paths, such as an upload endpoint |

**Tuning**

False positives are normal, especially for APIs. The tools, in order of preference:

1. **Exclusions:** tell specific rules to ignore a specific request attribute (a header, cookie, or body field such as `messages` or `prompt`).
2. **Rule overrides:** disable or change the action of an individual rule.
3. **Custom allow rules** for trusted sources.
4. Disabling a rule group, as a last resort.

Never fix a false positive by switching the whole WAF to Detection in production.

**Limits that matter**

- **Request body inspection** has a size limit (configurable up to a maximum, on the order of megabytes), with a separate **maximum request body size** and **file upload** limit. Requests beyond the inspection limit can be passed uninspected or blocked depending on settings.
- Body inspection can be disabled per policy when another layer handles it.

### WAF for AI applications

Chat and agent APIs stress WAFs in particular ways:

| Issue | Handling |
|---|---|
| Prompts contain SQL, code, HTML and shell text that match injection rules | Add **exclusions** for the prompt/message body fields on the relevant rules, scoped per-URI to the chat endpoint |
| Large request bodies (documents, images, long context) | Raise body size limits for that path; consider direct-to-storage uploads with SAS or user delegation instead of sending files through the gateway |
| Streaming responses | Disable response buffering on the gateway for the route |
| Long-running requests | Raise the backend request timeout |
| Abuse and cost attacks on a paid model endpoint | **Rate-limit** rules per client; bot protection; require authentication before any model call |
| Prompt injection | A WAF does **not** understand it. That is the job of Azure AI Content Safety Prompt Shields and application design (topic 23) |

### Application Gateway for Containers

A separate, newer service for AKS (and other Kubernetes) workloads.

| | Application Gateway | Application Gateway for Containers |
|---|---|---|
| Configuration | ARM resources | Kubernetes **Gateway API** or Ingress resources, via the ALB controller |
| Update speed | Minutes | Near real time |
| Features | Full feature set | Traffic splitting, mTLS, header rewrite, WAF support, autoscaling |
| Replaces | – | Application Gateway Ingress Controller (AGIC) |

Use it when application teams should define routing as part of their Kubernetes manifests.

## How it works on Azure

### Typical placement

```text
Internet ──► [Front Door, optional] ──► Application Gateway (WAF_v2) in snet-appgw
                                              │  end-to-end TLS
                                              ▼
                                   Container Apps / App Service / AKS (internal ingress)
                                              │
                                              ▼
                              Private endpoints: Foundry, Search, Storage, …
```

For an internal application, the gateway has only a private frontend and is reached over ExpressRoute, VPN or peering.

### Backends on App Service and Container Apps

Both are multi-tenant platforms that route by host name. Recommended setup:

1. Give the app a **custom domain** matching the public name and bind a certificate.
2. Make the app **private**: an internal Container Apps environment, or a private endpoint on App Service with public access disabled.
3. Point the gateway's backend pool at the app's FQDN or private IP, with the custom domain as the host name and a custom probe using the same host.
4. Do not rely on *pick host name from backend target* in production, since the app then sees its default domain and generates wrong absolute URLs.

### Monitoring

Send diagnostics to Log Analytics:

| Log | Shows |
|---|---|
| **Access log** | Every request: client IP, URL, status, latency, backend chosen, TLS details |
| **Firewall log** | Every WAF match: rule ID, message, matched data, action |
| **Performance / metrics** | Capacity units, backend health, response time, failed requests |

Useful queries:

```kusto
// Top WAF rules firing, to guide tuning
AGWFirewallLogs
| where TimeGenerated > ago(24h)
| summarize hits = count() by RuleId, Message, Action
| order by hits desc

// Requests blocked for one URI
AGWFirewallLogs
| where Action == "Blocked" and RequestUri has "/api/chat"
| project TimeGenerated, ClientIp, RuleId, Message, DetailedMessage, DetailedData

// Backend latency and errors
AGWAccessLogs
| where TimeGenerated > ago(1h)
| summarize p95 = percentile(TimeTaken, 95), errors = countif(HttpStatus >= 500) by bin(TimeGenerated, 5m), Host
```

Table and column names depend on whether resource-specific or legacy diagnostics tables are used.

### Diagnosing 502 Bad Gateway

The gateway's most common error. Causes, in rough order of likelihood:

1. **Backend unhealthy**: check *Backend health* in the portal; it states why.
2. **NSG or UDR** blocking the gateway from reaching the backend, or blocking required management ports.
3. **Host name or certificate mismatch** on HTTPS backends.
4. **Request timeout** shorter than the backend's response time.
5. **DNS**: the gateway cannot resolve the backend FQDN (it uses the VNet's DNS settings).
6. **Empty backend pool** or all instances failing the probe.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Application Gateway or Front Door | Application Gateway for regional and private entry; Front Door for global and edge. Both only with reason |
| Shared or per-workload gateway | Shared (multi-site listeners, per-site WAF policies) saves cost for many small apps but couples their change cycles. Per-workload for isolation and independent ownership |
| WAF at the gateway or at Front Door | At whichever is the first layer 7 hop exposed to clients. Avoid duplicating managed rules at two layers |
| Minimum instance count | At least 2 in production for availability and to absorb bursts |
| End-to-end TLS | Yes, unless there is a measured reason not to |
| Rule set version | Current DRS; plan upgrades, as newer versions change scoring and rule IDs |
| Detection first | Run new policies in Detection, tune from logs, then switch to Prevention |

## Hands-on

A WAF_v2 gateway costs a few hundred currency units a month if left running; delete it when done.

```bash
RG=rg-agw-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Network and a simple backend (a public sample site stands in for an app)
az network vnet create -g $RG -n vnet-agw --address-prefixes 10.30.0.0/16 \
  --subnet-name snet-appgw --subnet-prefixes 10.30.0.0/24 -o none
az network public-ip create -g $RG -n pip-agw --sku Standard --zone 1 2 3 -o none

# 2. WAF policy in Detection mode with the default managed rules
az network application-gateway waf-policy create -g $RG -n waf-lab -o none
az network application-gateway waf-policy policy-setting update -g $RG --policy-name waf-lab \
  --mode Detection --state Enabled --request-body-check true -o none

# 3. Gateway
az network application-gateway create -g $RG -n agw-lab -l $LOC \
  --sku WAF_v2 --capacity 1 --zones 1 2 3 \
  --vnet-name vnet-agw --subnet snet-appgw --public-ip-address pip-agw \
  --waf-policy waf-lab --priority 100 \
  --servers example.com --http-settings-protocol Https --http-settings-port 443 -o none

# 4. Fix host header for the sample backend and inspect health
az network application-gateway http-settings update -g $RG --gateway-name agw-lab -n appGatewayBackendHttpSettings \
  --host-name-from-backend-pool true -o none
az network application-gateway show-backend-health -g $RG -n agw-lab \
  --query "backendAddressPools[0].backendHttpSettingsCollection[0].servers[].{Address:address, Health:health}" -o table

# 5. Send a benign and a suspicious request
IP=$(az network public-ip show -g $RG -n pip-agw --query ipAddress -o tsv)
curl -s -o /dev/null -w "%{http_code}\n" "http://$IP/"
curl -s -o /dev/null -w "%{http_code}\n" "http://$IP/?id=1%27%20OR%20%271%27=%271"

# 6. Switch to Prevention and repeat the suspicious request (expect 403)
az network application-gateway waf-policy policy-setting update -g $RG --policy-name waf-lab --mode Prevention -o none
sleep 60
curl -s -o /dev/null -w "%{http_code}\n" "http://$IP/?id=1%27%20OR%20%271%27=%271"

# 7. Add a rate-limit custom rule and a geo rule in the portal, then review
az network application-gateway waf-policy custom-rule list -g $RG --policy-name waf-lab -o table

az group delete -n $RG --yes --no-wait
```

In the portal, also open *Backend health*, *Rewrites* and the WAF policy's *Managed rules → Exclusions* to see how tuning is expressed. With diagnostics enabled, run the firewall log query above.

## Security, cost and reliability notes

- **Security:** make the gateway the **only** path to the backend. Private backends and NSGs that allow only the gateway subnet.
- **Security:** a WAF reduces exposure to known attack patterns. It does not replace secure code, authentication or input validation, and it does not address prompt injection.
- **Security:** keep managed rule sets current and review firewall logs; a WAF nobody watches is running in the dark.
- **Cost:** fixed hourly cost plus capacity units. WAF_v2 costs more than Standard_v2. A shared gateway spreads the fixed cost.
- **Reliability:** zone-redundant, minimum two instances, and autoscale headroom. Scale-out takes minutes, so pre-scale before known events.
- **Reliability:** align the gateway's request timeout with the backend's and with any proxy in front.

## Common mistakes

- Undersized gateway subnet.
- NSG blocking `GatewayManager` ports, leaving the gateway in a failed state.
- Applying a default-route UDR to the gateway subnet without checking the constraints.
- TLS terminated at the gateway with plain HTTP onward, in a design that claims encryption in transit.
- Host header mismatches with App Service and Container Apps.
- Going straight to Prevention with no tuning, then disabling the WAF when it blocks real users.
- Leaving a policy in Detection indefinitely.
- Backends still reachable from the internet.
- Default 20 to 30 second timeouts on a chat endpoint that streams for a minute.
- Expecting the WAF to stop prompt injection.

## Check yourself

1. Name the components a request passes through in Application Gateway.
2. What is the difference between TLS termination and end-to-end TLS?
3. What do Detection and Prevention modes do?
4. In what order are custom and managed WAF rules evaluated?
5. How should you handle a WAF false positive on a chat API's message body?
6. What subnet and NSG requirements does Application Gateway v2 have?
7. List four common causes of a 502 from the gateway.
8. When would you choose Application Gateway for Containers?

<details>
<summary>Answers</summary>

1. Frontend IP, listener, routing rule, backend settings, backend pool (with a health probe determining which targets are used).
2. Termination decrypts at the gateway and forwards plain HTTP. End-to-end decrypts for inspection, then re-encrypts to the backend.
3. Detection logs matches without blocking; Prevention blocks according to the rules.
4. Custom rules first, by priority, then managed rules.
5. Add a targeted exclusion for that request attribute on the offending rules, scoped to the endpoint, rather than disabling the WAF or whole rule groups.
6. A dedicated subnet (a `/24` recommended), with inbound allowed from `GatewayManager` on 65200–65535, from `AzureLoadBalancer`, and from clients on the listener ports.
7. Unhealthy backend, NSG or UDR blocking, host name or certificate mismatch, request timeout, DNS resolution failure, empty pool.
8. For AKS workloads where routing should be defined with Kubernetes Gateway API or Ingress resources and updated quickly.

</details>

## References

- [What is Azure Application Gateway?](https://learn.microsoft.com/azure/application-gateway/overview)
- [Application Gateway components](https://learn.microsoft.com/azure/application-gateway/application-gateway-components)
- [Application Gateway infrastructure configuration](https://learn.microsoft.com/azure/application-gateway/configuration-infrastructure)
- [Overview of TLS termination and end-to-end TLS](https://learn.microsoft.com/azure/application-gateway/ssl-overview)
- [What is Azure Web Application Firewall on Application Gateway?](https://learn.microsoft.com/azure/web-application-firewall/ag/ag-overview)
- [WAF DRS and CRS rule groups and rules](https://learn.microsoft.com/azure/web-application-firewall/ag/application-gateway-crs-rulegroups-rules)
- [WAF exclusion lists](https://learn.microsoft.com/azure/web-application-firewall/ag/application-gateway-waf-configuration)
- [Troubleshoot bad gateway errors](https://learn.microsoft.com/azure/application-gateway/application-gateway-troubleshooting-502)
- [Preserve the original HTTP host name between a reverse proxy and its backend](https://learn.microsoft.com/azure/architecture/best-practices/host-name-preservation)
- [What is Application Gateway for Containers?](https://learn.microsoft.com/azure/application-gateway/for-containers/overview)
