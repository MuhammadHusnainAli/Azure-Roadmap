# Application and API Security

## Why it matters

Applications and APIs are where attackers meet your business logic. Network controls can't stop a valid, authenticated request that reads another customer's order because the API forgot to check ownership, or a token that's accepted without validating its audience, or a secret committed to a repository. Most serious application breaches come from **broken authorisation, weak authentication, exposed secrets and excessive data exposure**, all design and implementation issues. This file brings together the Azure patterns for authenticating users and services, handling secrets, protecting web apps with WAF and securing APIs with API Management and Defender for APIs, mapped to the OWASP API Security Top 10.

## Key concepts

### Authentication patterns

| Scenario | Pattern |
|---|---|
| **Users to web app** | OpenID Connect with Entra ID (workforce) or Entra External ID (customers); App Service/Container Apps built-in authentication ("Easy Auth") for simple cases |
| **SPA / mobile to API** | Authorisation code flow with PKCE; API validates access tokens (issuer, audience, signature, expiry, scopes/roles) |
| **API to downstream API as the user** | **On-behalf-of (OBO)** flow, preserving user identity and permissions |
| **Service to service (Azure)** | **Managed identity** with Entra tokens; no secrets |
| **Service from outside Azure** (CI/CD, other clouds, Kubernetes) | **Workload identity federation** (OIDC trust) instead of client secrets |
| **Partner/B2B APIs** | OAuth client credentials with certificates, or mutual TLS; API Management subscription keys only as an additional identifier, not authentication |

Authorise in the API using **scopes** (delegated) and **app roles** (application), and enforce **object-level** checks (does this caller own this resource?).

### Secrets hygiene

| Practice | How |
|---|---|
| Eliminate secrets | Managed identities, workload identity federation, Entra auth for Azure services (disable local/key auth) |
| Store what remains centrally | Key Vault with RBAC; Key Vault references in App Service, Functions, Container Apps, App Configuration (file 03) |
| Prevent leaks | GitHub secret scanning and push protection; pre-commit hooks (file 09) |
| Rotate | Automated rotation; short-lived credentials; certificates over secrets |
| Detect misuse | Defender for Key Vault, Entra workload identity risk detections |

### OWASP API Security Top 10 (2023) and Azure mitigations

| Risk | Azure mitigations |
|---|---|
| **API1 Broken object level authorisation** | Object ownership checks in code; APIM `validate-jwt` claims plus backend authorisation |
| **API2 Broken authentication** | Entra tokens; APIM `validate-jwt`/`validate-azure-ad-token`; no API keys as sole auth; mTLS for partners |
| **API3 Broken object property level authorisation** | Response shaping; APIM `validate-content`/response transformation to strip fields; avoid mass assignment |
| **API4 Unrestricted resource consumption** | APIM rate limits and quotas (`rate-limit-by-key`, `quota-by-key`), request size limits, timeouts |
| **API5 Broken function level authorisation** | App roles; separate admin APIs; APIM products and policies per audience |
| **API6 Unrestricted access to sensitive business flows** | Bot protection (WAF), CAPTCHA, business-level throttling, anomaly detection |
| **API7 Server-side request forgery** | Validate and allow-list outbound URLs; egress firewall; no metadata endpoint access from user input |
| **API8 Security misconfiguration** | Policy-enforced TLS, CORS restrictions, disable unused endpoints and verbose errors |
| **API9 Improper inventory management** | API Center inventory, Defender for APIs discovery of unused/unauthenticated APIs, versioning and retirement |
| **API10 Unsafe consumption of APIs** | Validate third-party responses; treat them as untrusted; timeouts and circuit breakers |

### Web Application Firewall

| Option | Use |
|---|---|
| **Front Door WAF** | Global edge protection with managed rule sets (DRS), bot protection, rate limiting, geo-filtering |
| **Application Gateway WAF** | Regional, in-VNet protection; private backends |
| **Modes** | Detection while tuning, then Prevention; exclusions for false positives with justification |

### Defender for APIs

Defender for Cloud plan for APIs published in **API Management**: inventory, security findings (external, unused, unauthenticated APIs), posture recommendations, sensitive data classification, runtime threat detection including OWASP API Top 10 threats, integration with the cloud security graph and SIEM.

## How it works on Azure

API Management inbound policy validating Entra tokens, enforcing a scope and rate-limiting per caller:

```xml
<policies>
  <inbound>
    <base />
    <validate-azure-ad-token tenant-id="{{tenant-id}}" header-name="Authorization"
        failed-validation-httpcode="401" failed-validation-error-message="Unauthorised">
      <client-application-ids>
        <application-id>{{orders-spa-client-id}}</application-id>
      </client-application-ids>
      <audiences><audience>api://orders-api</audience></audiences>
      <required-claims>
        <claim name="scp" match="any" separator=" "><value>Orders.Read</value></claim>
      </required-claims>
    </validate-azure-ad-token>
    <rate-limit-by-key calls="100" renewal-period="60"
        counter-key="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Subject)" />
    <set-header name="X-Forwarded-User" exists-action="override">
      <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("oid", ""))</value>
    </set-header>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
</policies>
```

The backend must still verify that the caller (`oid`) owns the requested order (API1).

Reference architecture:

```text
Client ─► Front Door (WAF, bot protection, DDoS) ─► APIM (token validation, rate limits, schema validation)
        ─► Private backend (App Service / Container Apps with managed identity, VNet-integrated)
        ─► Data (SQL, Cosmos DB via private endpoints, Entra auth, no keys)
Secrets: Key Vault references; Monitoring: App Insights, APIM logs, WAF logs → Sentinel; Defender for APIs on APIM
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Easy Auth vs in-code auth** | Easy Auth for simple apps; in-code (MSAL, Microsoft.Identity.Web) for fine-grained authorisation and OBO. |
| **Gateway vs app validation** | Validate at both: gateway for coarse checks and throttling, app for object-level authorisation. |
| **WAF placement** | Front Door for global apps; App Gateway for regional/private; avoid double WAF unless needed. |
| **API keys** | Never as the only authentication for sensitive APIs. |
| **Local auth on Azure services** | Disable (Storage shared keys, Cosmos DB keys, Service Bus SAS, Azure OpenAI keys) and use Entra. |
| **Inventory** | Register every API in API Center; retire unused versions. |

## Hands-on

1. Protect an API with Entra: register API and client apps, define a scope and an app role.
2. Front it with APIM using the policy above; test with valid and invalid tokens.
3. Implement an object-level ownership check in the backend; write a test proving one user can't read another's order.
4. Disable shared key access on a storage account and switch the app to managed identity.
5. Enable Defender for APIs and review findings; put Front Door WAF in Detection, review logs, then switch to Prevention.

## Security, cost and reliability notes

- **Authorisation bugs are invisible to scanners**: test them explicitly (unit and integration tests, pen tests).
- **Token handling**: validate audience and issuer; don't log tokens; short lifetimes.
- **WAF false positives** can break legitimate flows; tune in Detection mode first.
- **Cost**: Front Door Premium, APIM tiers, Defender for APIs; size by traffic and risk.
- **Resilience**: throttling and circuit breakers protect backends during attacks and spikes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on APIM subscription keys for authentication | Entra tokens |
| Checking authentication but not object ownership | Object-level authorisation |
| Secrets in app settings or code | Managed identity, Key Vault references |
| WAF permanently in Detection mode | Tune, then Prevention |
| No API inventory | API Center and Defender for APIs |

## Check yourself

1. Which flow lets an API call another API as the user?
2. How should workloads outside Azure authenticate without secrets?
3. Name five OWASP API Security Top 10 risks and an Azure mitigation for each.
4. What does `validate-azure-ad-token` check, and what must the backend still do?
5. What does Defender for APIs provide?
6. Why disable local authentication on Azure services?

<details>
<summary>Answers</summary>

1. The OAuth 2.0 on-behalf-of (OBO) flow.
2. Workload identity federation (OIDC trust between the external identity provider and Entra).
3. For example: API1 BOLA → ownership checks; API2 broken auth → Entra token validation; API4 resource consumption → APIM rate limits and quotas; API7 SSRF → URL allow-lists and egress firewall; API9 inventory → API Center and Defender for APIs.
4. Token signature, tenant, audience, client application IDs and required claims; the backend must still enforce object-level and business authorisation.
5. API inventory, security findings, posture recommendations, sensitive data classification, runtime threat detection (OWASP API Top 10), CSPM graph and SIEM integration, for APIs in API Management.
6. Keys and SAS tokens are long-lived shared secrets that bypass identity controls; Entra auth enables RBAC, Conditional Access and auditing.

</details>

## References

- [Mitigate OWASP API Security Top 10 threats with API Management](https://learn.microsoft.com/azure/api-management/mitigate-owasp-api-threats)
- [About Microsoft Defender for APIs](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-apis-introduction)
- [Azure Web Application Firewall](https://learn.microsoft.com/azure/web-application-firewall/overview)
- [On-behalf-of flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [Well-Architected: application secrets](https://learn.microsoft.com/azure/well-architected/security/application-secrets)
- [validate-jwt policy](https://learn.microsoft.com/azure/api-management/validate-jwt-policy)
