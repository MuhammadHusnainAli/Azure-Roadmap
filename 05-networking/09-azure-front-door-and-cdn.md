# Azure Front Door and CDN

## Why it matters

Azure Front Door is the global entry point for internet-facing HTTP applications: one hostname that reaches users through Microsoft's edge network, terminates TLS close to them, blocks attacks before they reach your region, caches what can be cached, and fails over between regions in seconds. It is the default answer for multi-region web applications, and increasingly for single-region ones that want edge security.

## Key concepts

### What it is

A layer 7 reverse proxy that runs on Microsoft's global edge network (well over a hundred points of presence). It is a **global, non-regional** service; you do not deploy it into a virtual network.

```text
User ──► nearest edge PoP (anycast) ──► Microsoft backbone ──► origin in region A or B
          TLS termination, WAF,
          cache, rules, routing
```

Why this helps:

| Mechanism | Benefit |
|---|---|
| **Anycast** | Users connect to the nearest edge location using one IP range |
| **Split TCP / TLS at the edge** | Short client connections; warm, reused connections from edge to origin |
| **Backbone transit** | Traffic rides Microsoft's network instead of the public internet for most of the path |
| **Health-based routing** | Unhealthy origins are removed in seconds, with no DNS TTL to wait for |
| **Edge enforcement** | Attacks are absorbed and filtered far from your application |

### Tiers

| | **Standard** | **Premium** |
|---|---|---|
| Content delivery, caching, compression | Yes | Yes |
| Rules engine | Yes | Yes |
| Custom domains, managed certificates | Yes | Yes |
| WAF custom rules | Yes | Yes |
| WAF **managed rule sets** (DRS) | No | Yes |
| **Bot protection** | No | Yes |
| **Private Link to origins** | No | Yes |
| Security reports and enhanced analytics | Basic | Yes |

Premium is the choice for production applications that need managed WAF rules or private origins.

**Retirements:** *Azure Front Door (classic)* retires on 31 March 2027, and *Azure CDN Standard from Microsoft (classic)* on 30 September 2027. Azure CDN from Edgio (Verizon) is already gone. All new work uses Front Door Standard or Premium; migrate classic profiles with the built-in migration tooling.

### Building blocks

| Component | Role |
|---|---|
| **Profile** | The top-level resource and tier |
| **Endpoint** | A hostname `<name>-<hash>.z01.azurefd.net` |
| **Custom domain** | Your hostname mapped by CNAME (or alias record at the apex), with an Azure-managed or your own certificate |
| **Origin** | A backend: App Service, Container Apps, Storage, Application Gateway, API Management, a public IP, or any public hostname; or a private resource through Private Link |
| **Origin group** | Origins that serve the same content, with health probe and load-balancing settings |
| **Route** | Maps domains and path patterns to an origin group, with protocol, caching and rule set settings |
| **Rule set** | Rules engine logic attached to routes |
| **Security policy** | Associates a WAF policy with domains |

### Routing to origins

Within an origin group, selection uses:

1. **Health:** probes mark origins healthy or not.
2. **Priority:** lower number first. Equal priority means active-active; different priorities mean active-passive.
3. **Latency sensitivity:** among healthy same-priority origins, those within a configurable latency band of the fastest are eligible.
4. **Weight:** traffic is distributed across eligible origins by weight.

| Goal | Configuration |
|---|---|
| Active-passive failover | Primary priority 1, secondary priority 2 |
| Active-active, nearest region | Same priority, equal weight, default latency sensitivity |
| Gradual migration or canary | Same priority, weights 90 and 10 |
| Sticky sessions | Enable session affinity (cookie-based) |

Route matching is by the most specific domain and path pattern (`/api/*`, `/static/*`, `/*`).

### Caching

Enabled per route.

| Setting | Options |
|---|---|
| **Query string behaviour** | Ignore, use all, include specific, or exclude specific parameters in the cache key |
| **Compression** | Brotli or gzip at the edge for eligible content types |
| **Cache duration** | Honour origin `Cache-Control` headers, or override with rules |
| **Purge** | Invalidate by path or wildcard |

Cache static assets aggressively with versioned file names. **Do not cache** personalised or authenticated API responses, or model completions, unless you have deliberately designed a cache key that makes that safe.

### Rules engine

Rule sets run at the edge on requests and responses.

| Conditions (examples) | Actions (examples) |
|---|---|
| Request path, method, header, query string, cookie | Redirect (for example HTTP to HTTPS, apex to `www`) |
| Client IP, geography, device type | URL rewrite |
| Request scheme, host name, file extension | Add, overwrite or delete request and response headers |
| Server variables | Override the origin group or cache settings for the request |

Common uses: security headers (`Strict-Transport-Security`, `Content-Security-Policy`), canonical redirects, routing `/api/*` to a different origin, long cache lifetimes for hashed assets.

### Web Application Firewall at the edge

Front Door's WAF runs at the edge, so malicious traffic never reaches your region.

| Feature | Notes |
|---|---|
| **Managed rules** (Premium) | Microsoft Default Rule Set with anomaly scoring and threat intelligence rules |
| **Bot Manager** (Premium) | Good, bad and unknown bot categories |
| **Custom rules** | Match on IP, geography, headers, URI, body, and more |
| **Rate limiting** | Per client IP (or socket address) thresholds over 1 or 5 minutes |
| **Geo-filtering** | Allow or block by country |
| **Challenges** | JavaScript challenge and CAPTCHA to separate people from bots |
| **Modes** | Detection and Prevention |
| **Exclusions and overrides** | Tuning for false positives |

The same tuning discipline as Application Gateway applies (file 08), including exclusions for prompt bodies on AI endpoints. Front Door also provides built-in protection against network-layer DDoS attacks at the edge.

### Securing the origin

Front Door only protects you if nothing can reach the origin around it.

| Method | How | Strength |
|---|---|---|
| **Private Link origin** (Premium) | Front Door connects to the origin over a private endpoint that you approve. The origin has no public access. Supported for App Service, Storage, internal load balancers, Application Gateway, API Management, Container Apps and others | Strongest |
| **Service tag + header check** | Allow only the `AzureFrontDoor.Backend` service tag in the origin's access restrictions or NSG, **and** validate the `X-Azure-FDID` request header equals your profile's ID | Good. The header check is essential because the tag covers every Front Door customer |
| **Managed identity to origin** | Front Door authenticates to origins that support Entra authentication using its managed identity | Complements the above |
| Header secret only | A shared secret header set by the rules engine and checked by the app | Weak alone |

### Protocols and limits to know

| Topic | Detail |
|---|---|
| Protocols to clients | HTTP/1.1, HTTP/2, and HTTP/3 (QUIC) where enabled; TLS 1.2 and 1.3 |
| To origins | HTTP/1.1 |
| **WebSockets** | Supported on Standard and Premium |
| **Origin response timeout** | Configurable per profile, with a maximum of a few minutes (240 seconds). Requests exceeding it return an error |
| **Server-sent events / streaming** | Works when the origin sends data steadily. Confirm chunked responses are passed through for your route and that caching and compression are not buffering them |
| Request body size | Inspected by WAF up to a limit; very large uploads are better sent directly to storage |
| Propagation | Configuration changes take several minutes to reach all edge locations. Plan deployments accordingly |

### Front Door as a CDN

Front Door Standard and Premium are Microsoft's content delivery network. For static sites and assets: put a storage account static website or blob container behind a route with caching enabled, use a custom domain and managed certificate, and set long cache lifetimes on versioned files.

## How it works on Azure

### Multi-region active-active web application

```text
                    ┌──────────── Azure Front Door Premium ────────────┐
   Users ─────────► │  custom domain · WAF · caching · rules          │
                    └───────────┬──────────────────────┬───────────────┘
                     Private Link│                     │Private Link
                     ┌──────────▼────────┐   ┌─────────▼─────────┐
                     │ Region A           │   │ Region B           │
                     │ Container Apps     │   │ Container Apps     │
                     │ + data + AI        │   │ + data + AI        │
                     └────────────────────┘   └────────────────────┘
```

Origin group with both regions at priority 1; a health probe on `/healthz`; WAF policy in Prevention with rate limiting on `/api/chat`.

The harder part of multi-region is state: databases, search indexes and conversation history must be available in both regions. Front Door solves the routing, not the data (topic 28).

### Front Door with Application Gateway

Use both only for a reason:

| Reason | Layout |
|---|---|
| Global entry plus private regional routing to many internal apps | Front Door → Application Gateway (Private Link or locked to the service tag) → apps |
| Regional features Front Door lacks (certain mTLS-to-backend or routing needs) | Same |

Enable managed WAF rules at **one** layer, usually Front Door, and use the gateway for routing.

### Front Door in front of an AI application

| Concern | Guidance |
|---|---|
| Streaming chat | Verify end-to-end with your client; disable caching and compression on the route |
| Long generations | Raise the origin response timeout; design the app to stream early so the connection stays active |
| Real-time voice | WebSockets are supported; WebRTC media does not traverse Front Door, so the signalling and media paths differ |
| Cost abuse | Rate-limit rules and bot protection on model-backed endpoints; require sign-in |
| Static front end | Serve from the same profile with caching, with `/api/*` routed to the application origin |
| Data residency | Front Door is a global service; TLS terminates at edge locations worldwide. For strict in-geography processing requirements, assess whether that is acceptable or use a regional entry point |

### Monitoring

| Source | Shows |
|---|---|
| **Access logs** | Every request: edge location, origin, cache status, latency, status codes |
| **WAF logs** | Rule matches and actions |
| **Health probe logs** | Origin health transitions |
| **Metrics** | Request count, latency (total and origin), error percentages, origin health percentage, cache hit ratio |
| **Security and traffic reports** (Premium) | Built-in dashboards |

Front Door returns an `X-Azure-Ref` header on every response. Capture it in bug reports; it identifies the request in logs and to support.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Standard or Premium | Premium for managed WAF, bot protection and Private Link origins |
| Front Door for one region | Worth it for edge WAF, managed certificates and future expansion. Skip it for internal applications |
| Active-active or active-passive | Active-active uses capacity in both regions and proves failover continuously, but requires multi-region data. Active-passive is simpler and may have a cold standby problem |
| Private Link origin or service tag | Private Link where supported |
| Caching | Static assets yes. Dynamic and personalised content no, without careful cache-key design |
| Front Door or Traffic Manager | Front Door for HTTP. Traffic Manager for other protocols or as a DNS layer above |
| Single global entry point risk | Front Door is itself a dependency. For the most critical workloads consider an alternate path (a second CDN or direct regional entry) switched by Traffic Manager, and weigh that complexity honestly |

## Hands-on

```bash
RG=rg-afd-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. Two simple origins in different regions (static websites on storage)
for R in swedencentral westeurope; do
  ST=stafd${R:0:3}$RANDOM
  az storage account create -g $RG -n $ST -l $R --sku Standard_LRS -o none
  az storage blob service-properties update --account-name $ST --static-website --index-document index.html --auth-mode login -o none
  echo "<h1>Hello from $R</h1>" > index.html
  az storage blob upload --account-name $ST -c '$web' -f index.html -n index.html --content-type text/html --auth-mode login --overwrite -o none
  echo "$R $(az storage account show -g $RG -n $ST --query primaryEndpoints.web -o tsv)"
done
```

Uploading requires *Storage Blob Data Contributor* on the accounts. Note the two web endpoint hostnames, then:

```bash
ORIGIN_A=<host-from-region-1>     # e.g. stafdswe123.z1.web.core.windows.net
ORIGIN_B=<host-from-region-2>

# 2. Front Door profile, endpoint, origin group and origins
az afd profile create -g $RG --profile-name afd-lab --sku Standard_AzureFrontDoor -o none
az afd endpoint create -g $RG --profile-name afd-lab --endpoint-name lab-$RANDOM --enabled-state Enabled -o none
EP=$(az afd endpoint list -g $RG --profile-name afd-lab --query "[0].name" -o tsv)
az afd origin-group create -g $RG --profile-name afd-lab --origin-group-name og-web \
  --probe-request-type GET --probe-protocol Https --probe-interval-in-seconds 30 --probe-path / \
  --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50 -o none
az afd origin create -g $RG --profile-name afd-lab --origin-group-name og-web --origin-name a \
  --host-name $ORIGIN_A --origin-host-header $ORIGIN_A --priority 1 --weight 1000 --enabled-state Enabled --https-port 443 -o none
az afd origin create -g $RG --profile-name afd-lab --origin-group-name og-web --origin-name b \
  --host-name $ORIGIN_B --origin-host-header $ORIGIN_B --priority 2 --weight 1000 --enabled-state Enabled --https-port 443 -o none

# 3. Route with HTTPS redirect and caching
az afd route create -g $RG --profile-name afd-lab --endpoint-name $EP --route-name default \
  --origin-group og-web --supported-protocols Http Https --https-redirect Enabled \
  --forwarding-protocol HttpsOnly --link-to-default-domain Enabled \
  --enable-caching true --query-string-caching-behavior IgnoreQueryString -o none

# 4. Test (allow several minutes for propagation)
HOST=$(az afd endpoint show -g $RG --profile-name afd-lab --endpoint-name $EP --query hostName -o tsv)
curl -sI https://$HOST/ | grep -iE "HTTP/|x-cache|x-azure-ref"
curl -s  https://$HOST/

# 5. Failover: disable origin a and watch traffic move to b (purge the cache first)
az afd origin update -g $RG --profile-name afd-lab --origin-group-name og-web --origin-name a --enabled-state Disabled -o none
az afd endpoint purge -g $RG --profile-name afd-lab --endpoint-name $EP --content-paths '/*'
sleep 120; curl -s https://$HOST/

az group delete -n $RG --yes --no-wait
```

Observe the `x-cache` header (`TCP_MISS`, then `TCP_HIT`) and the `x-azure-ref` value. In the portal, attach a WAF policy with a rate-limit rule and review the rule set editor.

## Security, cost and reliability notes

- **Security:** an origin reachable directly from the internet makes the WAF optional for attackers. Lock it down and verify from outside.
- **Security:** managed certificates remove renewal risk for custom domains. Guard the DNS records; a dangling CNAME to a deleted endpoint enables subdomain takeover.
- **Security:** enforce HTTPS and set HSTS in a rule set.
- **Cost:** a base fee per profile, plus requests and data transfer. Premium's base fee is substantially higher than Standard's. Caching reduces origin egress and compute.
- **Reliability:** probes run from many edge locations, so a short interval multiplies requests to your origin. Use a lightweight health path and a sensible interval.
- **Reliability:** configuration changes propagate over minutes and a bad change is global. Manage with IaC and test in a separate profile.

## Common mistakes

- Origins left publicly reachable.
- Allowing the `AzureFrontDoor.Backend` tag without checking `X-Azure-FDID`.
- Starting new work on classic Front Door or classic CDN.
- Caching authenticated or personalised responses.
- Forgetting to set the origin host header so the origin does not recognise the request.
- Health probes against an expensive endpoint from a hundred edge locations.
- Expecting instant configuration changes.
- Default origin timeout cutting off long model responses.
- Managed WAF rules duplicated at Front Door and Application Gateway.
- A multi-region front end over a single-region database.

## Check yourself

1. Why does Front Door fail over faster than DNS-based routing?
2. Name three capabilities that require the Premium tier.
3. How does Front Door choose an origin within an origin group?
4. What are the two main ways to stop traffic bypassing Front Door, and which is stronger?
5. Why is checking the `X-Azure-FDID` header necessary when using the service tag?
6. What should and should not be cached?
7. What should you verify before putting Front Door in front of a streaming chat API?
8. Which Front Door and CDN products are being retired?

<details>
<summary>Answers</summary>

1. It is in the data path and stops sending requests to an unhealthy origin within seconds, with no dependence on DNS TTLs or client caching.
2. Managed WAF rule sets, bot protection, and Private Link origins.
3. Healthy origins only, then lowest priority value, then those within the latency sensitivity band, then distribution by weight.
4. Private Link origins, and restricting to the `AzureFrontDoor.Backend` service tag with an `X-Azure-FDID` header check. Private Link is stronger.
5. The service tag includes every customer's Front Door; the header identifies requests from your specific profile.
6. Cache static, versioned assets. Do not cache personalised, authenticated or dynamically generated responses without a deliberately designed cache key.
7. That chunked responses stream through without buffering, caching and compression are off for the route, and the origin response timeout is long enough.
8. Azure Front Door (classic), retiring 31 March 2027, and Azure CDN Standard from Microsoft (classic), retiring 30 September 2027.

</details>

## References

- [What is Azure Front Door?](https://learn.microsoft.com/azure/frontdoor/front-door-overview)
- [Azure Front Door tier comparison](https://learn.microsoft.com/azure/frontdoor/standard-premium/tier-comparison)
- [Origins and origin groups](https://learn.microsoft.com/azure/frontdoor/origin)
- [Traffic routing methods to origin](https://learn.microsoft.com/azure/frontdoor/routing-methods)
- [Caching with Azure Front Door](https://learn.microsoft.com/azure/frontdoor/front-door-caching)
- [Secure your origin with Private Link in Azure Front Door Premium](https://learn.microsoft.com/azure/frontdoor/private-link)
- [Secure traffic to Azure Front Door origins](https://learn.microsoft.com/azure/frontdoor/origin-security)
- [Web Application Firewall on Azure Front Door](https://learn.microsoft.com/azure/web-application-firewall/afds/afds-overview)
- [Azure Front Door (classic) retirement FAQ](https://learn.microsoft.com/azure/frontdoor/classic-retirement-faq)
- [Mission-critical global content delivery](https://learn.microsoft.com/azure/architecture/guide/networking/global-web-applications/mission-critical-content-delivery)
