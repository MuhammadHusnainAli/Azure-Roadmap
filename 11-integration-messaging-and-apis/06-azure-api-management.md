# Azure API Management

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

API Management (APIM) is the front door for an organisation's APIs. One gateway authenticates callers, applies rate limits and transformations, routes to back ends wherever they run, and records analytics. A developer portal lets consumers discover and subscribe. APIM is also the basis of Azure's **AI gateway** (file 07) and can expose REST APIs as **MCP servers** for agents. Architects meet it in almost every enterprise design. Consultants need to explain tiers and cost, because the tier choice drives networking, scale and price.

## Key concepts

### Components

| Component | Role |
|---|---|
| **Gateway** (data plane) | Receives API calls, enforces policies, forwards to back ends, returns responses |
| **Management plane** | Azure Resource Manager and the portal, CLI, Bicep and Terraform: define APIs, products, policies, users |
| **Developer portal** | Customisable, auto-generated website where consumers read docs, try APIs and get subscription keys |

### Core entities

| Entity | Meaning |
|---|---|
| **API** | A set of operations mapped to a back end (imported from OpenAPI, WSDL, WADL, GraphQL, Azure resources, or created manually) |
| **Operation** | One method and URL template (for example `GET /orders/{id}`) |
| **Backend** | A reusable definition of a back-end service (URL, credentials, circuit breaker); can be grouped into **load-balanced pools** |
| **Product** | A bundle of one or more APIs offered to consumers with terms, approval rules and policies (for example "Partner – Bronze") |
| **Subscription** | Grants a consumer access via a **subscription key** (`Ocp-Apim-Subscription-Key` header). Scoped to a product, a single API, or all APIs |
| **Named value** | A configuration value (optionally a Key Vault reference) used in policies |
| **Version / revision** | **Versions** are breaking changes exposed side by side (`/v1`, `/v2`, header or query). **Revisions** are non-breaking, testable changes you can make current |

### Policies

Policies are XML statements that run in four sections, at five scopes:

```text
Scopes (inherit with <base/>): global → workspace → product → API → operation

<policies>
  <inbound>   ... runs on the request before the back end
  <backend>   ... controls forwarding (retry, set-backend-service)
  <outbound>  ... runs on the response
  <on-error>  ... runs when any section errors
</policies>
```

Commonly used policies:

| Category | Policies |
|---|---|
| Access | `validate-jwt`, `validate-azure-ad-token`, `check-header`, `ip-filter`, `authentication-managed-identity` |
| Traffic | `rate-limit`, `rate-limit-by-key`, `quota`, `quota-by-key`, `llm-token-limit` |
| Routing | `set-backend-service`, `forward-request`, `retry`, `return-response`, `send-request` |
| Transformation | `set-header`, `set-query-parameter`, `rewrite-uri`, `set-body`, `json-to-xml`, `xml-to-json` |
| Caching | `cache-lookup`, `cache-store`, `cache-lookup-value`, `llm-semantic-cache-lookup` / `-store` |
| Validation | `validate-content`, `validate-parameters`, `validate-headers` |
| Observability | `trace`, `emit-metric`, `log-to-eventhub`, `llm-emit-token-metric` |

Policy expressions use C# syntax inside `@( ... )` or `@{ ... }`, with access to `context` (request, response, subscription, user, variables).

## How it works on Azure

### Tiers (checked 2026-11)

| Feature | Consumption | Developer | Basic | Basic v2 | Standard | Standard v2 | Premium | Premium v2 |
|---|---|---|---|---|---|---|---|---|
| SLA | Yes | **No** (non-production) | Yes | Yes | Yes | Yes | Yes | Yes |
| Inbound private endpoint | No | Yes | Yes | No | Yes | Yes | Yes | Yes |
| VNet injection (full isolation) | No | Yes | No | No | No | No | Yes | Yes |
| Reach back ends in a VNet (outbound integration) | No | Yes | No | No | No | Yes | Yes | Yes |
| Multi-region deployment | No | No | No | No | No | No | **Yes** | No |
| Availability zones | No | No | No | No | No | No | Yes | Yes |
| Self-hosted gateway | No | Yes (1 node) | No | No | No | No | **Yes** | No |
| Workspaces | No | No | No | Yes | No | Yes | Yes | Yes |
| Workspace gateways (dedicated) | No | No | No | No | No | No | Yes | Yes |
| Developer portal | No | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Max scale units | Auto | 1 | 2 | 10 | 4 | 10 | 12 per region | 30 |
| Built-in cache | None | 10 MB | 50 MB | 250 MB | 1 GB | 1 GB | 5 GB | 5 GB |
| Backup and restore | No | Yes | Yes | No | Yes | No | Yes | No |

Notes:

- The **v2 tiers** (Basic v2, Standard v2, Premium v2) are GA. They run on a newer platform with faster provisioning and scaling. Currently unavailable in v2: multi-region deployment, self-hosted gateway, backup/restore, Git configuration, and upgrading in place from classic tiers (new instances only).
- **Standard v2** VNet integration is outbound only: the gateway stays publicly reachable unless you add a private endpoint. **Premium v2** supports simplified VNet injection without route tables or service endpoints.
- The **Consumption** tier scales automatically and bills per call, with no developer portal, no VNet and a cold start.

### Self-hosted gateway

A containerised version of the gateway that you run on Kubernetes, Docker, Azure Arc or other clouds. It stays connected to the APIM management plane for configuration and telemetry. Use it to keep API traffic close to on-premises or multicloud back ends while managing everything centrally. It's available in **Developer** (single node) and **Premium** only, and it's billed per gateway deployment. Workspaces can't use self-hosted gateways.

### Workspaces (federated API management)

Workspaces act like folders inside one APIM instance. Each team manages its own APIs, products, subscriptions and named values, with RBAC scoped to the workspace, while a platform team governs global policies and the shared developer portal.

| Runtime option | Availability | Isolation |
|---|---|---|
| Default managed gateway | v2 tiers | Shared capacity and configuration with other workspaces |
| **Workspace gateway** (separate Azure resource) | Premium, Premium v2 | Dedicated scale, hostname and optional VNet per gateway; up to 30 workspaces per gateway |

Constraints to know (checked 2026-11): workspaces don't support MCP servers, credential manager, external cache, synthetic GraphQL or self-hosted gateways. Workspace gateways don't support inbound private endpoints or custom hostnames (put Application Gateway or Front Door in front). Resource names must be unique across the whole instance. Global policies still run on workspace gateways.

### Deploy and configure (Bicep and CLI)

```bicep
resource apim 'Microsoft.ApiManagement/service@2024-05-01' = {
  name: 'apim-contoso-prod'
  location: location
  sku: { name: 'StandardV2', capacity: 1 }
  identity: { type: 'SystemAssigned' }
  properties: {
    publisherName: 'Contoso'
    publisherEmail: 'api-team@contoso.com'
  }
}
```

```bash
# Import an API from an OpenAPI document and add it to a product
az apim api import -g rg-api --service-name apim-contoso-prod \
  --api-id orders --path orders --specification-format OpenApi \
  --specification-url https://raw.githubusercontent.com/contoso/specs/main/orders.yaml
az apim product api add -g rg-api --service-name apim-contoso-prod --product-id partners --api-id orders
```

### A typical API policy

```xml
<policies>
  <inbound>
    <base />
    <!-- Accept only tokens from our tenant for this API's app registration -->
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
      <client-application-ids><application-id>{{partner-portal-client-id}}</application-id></client-application-ids>
      <audiences><audience>api://orders-api</audience></audiences>
    </validate-azure-ad-token>
    <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
    <!-- Call the back end with the APIM managed identity; no secrets -->
    <authentication-managed-identity resource="api://orders-backend" />
    <set-backend-service backend-id="orders-backend" />
  </inbound>
  <backend><base /></backend>
  <outbound>
    <base />
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
  <on-error><base /></on-error>
</policies>
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Tier for production | Standard v2 for most new public-facing or hybrid workloads; Premium v2 when you need full VNet injection and zones; classic Premium when you need multi-region or a self-hosted gateway; Consumption for low-volume, serverless APIs |
| Developer tier | Never for production: no SLA, single unit |
| Subscription keys vs OAuth | Keys identify the *consuming application* for metering and products; they aren't strong authentication. Combine them with Entra ID (`validate-azure-ad-token` / `validate-jwt`) for real authorisation |
| Workspaces vs multiple instances | Workspaces for federated teams sharing governance and a portal; separate instances for hard isolation (different regulatory scopes, lifecycles) |
| Self-hosted gateway | When latency, data residency or network constraints require the gateway near back ends outside Azure |
| Inbound exposure | Premium (v2) injected internal mode plus Application Gateway with WAF for internet exposure; or private endpoint plus Front Door Premium |
| Versions vs revisions | Versions for breaking changes consumers opt in to; revisions for safe, testable, non-breaking changes |

## Hands-on

1. Create a Basic v2 or Standard v2 instance (provisioning takes minutes; classic tiers can take 30–45 minutes).
2. Import the public Petstore OpenAPI (or your own Function app), test it in the portal test console, and inspect the trace.
3. Create a product "Partners" that requires a subscription, add the API, create a subscription, and call the API with `Ocp-Apim-Subscription-Key`.
4. Add `rate-limit-by-key` and confirm a 429 response with `Retry-After` when exceeded.
5. Enable the system-assigned managed identity, protect your back end with Easy Auth, and use `authentication-managed-identity` so APIM calls it without secrets.
6. Enable the developer portal and publish it. Sign in as a consumer and try the API.

## Security, cost and reliability notes

- **Security:** combine subscription keys with Entra ID token validation. Store secrets as Key Vault-backed named values. Use managed identity to reach back ends.
- **Security:** protect internet exposure with WAF (Application Gateway or Front Door). Enable **Defender for APIs** for threat detection (not available for APIs in workspaces).
- **Security:** remove back-end details from responses (`X-Powered-By`, stack traces) in `outbound` and `on-error`.
- **Cost:** classic Premium is expensive per unit and per region. Check whether Standard v2 or Premium v2 meets the networking needs first. Workspace gateways and self-hosted gateways are billed separately.
- **Reliability:** Premium (classic and v2) supports availability zones; only classic Premium supports multi-region gateways. Otherwise deploy two instances behind Front Door.
- **Reliability:** use backend circuit breakers and retries carefully. Retrying non-idempotent operations can duplicate side effects.
- **Operations:** manage APIM as code: APIOps (extract and publish), Bicep or Terraform. Avoid portal-only changes.

## Common mistakes

- Treating subscription keys as authentication.
- Using the Developer tier in production.
- Choosing classic Premium only for VNet access when Standard v2 integration or Premium v2 would do.
- Hard-coding secrets in policies instead of Key Vault named values and managed identity.
- Putting heavy business logic in policy expressions.
- Expecting multi-region or a self-hosted gateway on v2 tiers.
- Forgetting `<base />`, so global or product policies silently stop applying.

## Check yourself

1. What are the three main components of APIM?
2. What is the difference between a product and a subscription?
3. In which policy section does `validate-jwt` belong, and why?
4. Which tiers support a self-hosted gateway?
5. What is the difference between VNet integration in Standard v2 and VNet injection in Premium v2?
6. What problem do workspaces solve, and which tiers support them?
7. Versions or revisions for a breaking change?
8. Why is the Developer tier unsuitable for production?

<details>
<summary>Answers</summary>

1. The gateway (data plane), the management plane, and the developer portal.
2. A product bundles APIs with terms and policies; a subscription grants a specific consumer access (with keys) to a product, an API or all APIs.
3. `inbound`, because the token must be validated before the request reaches the back end.
4. Developer (single node) and classic Premium.
5. Standard v2 integration only lets the gateway reach back ends in a VNet (outbound); the gateway stays publicly reachable. Premium v2 injection places the gateway inside the VNet, isolating inbound and outbound traffic.
6. Federated API management: teams manage their own APIs with scoped RBAC in a shared instance with central governance. Supported in Basic v2, Standard v2, Premium and Premium v2; dedicated workspace gateways need Premium or Premium v2.
7. Versions.
8. It has no SLA and runs a single unit; it is meant for evaluation and non-production use.

</details>

## References

- [About API Management](https://learn.microsoft.com/azure/api-management/api-management-key-concepts)
- [Feature-based comparison of tiers](https://learn.microsoft.com/azure/api-management/api-management-features)
- [v2 tiers overview](https://learn.microsoft.com/azure/api-management/v2-service-tiers-overview)
- [Policies in API Management](https://learn.microsoft.com/azure/api-management/api-management-howto-policies)
- [Policy reference](https://learn.microsoft.com/azure/api-management/api-management-policies)
- [Workspaces](https://learn.microsoft.com/azure/api-management/workspaces-overview)
- [Self-hosted gateway overview](https://learn.microsoft.com/azure/api-management/self-hosted-gateway-overview)
- [Developer portal overview](https://learn.microsoft.com/azure/api-management/developer-portal-overview)
- [Networking options](https://learn.microsoft.com/azure/api-management/virtual-network-concepts)
