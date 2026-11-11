# Generative AI Gateway

## Why it matters

When several teams consume models, enterprises need a control point. It enforces quotas per team, balances load across deployments and regions, fails over when a back end is throttled, applies consistent security, logs usage and supports chargeback. Microsoft Foundry doesn't provide round-robin load balancing or circuit breaking across deployments by itself, so the recommended pattern is a **generative AI gateway** built on **Azure API Management**. In this capstone you build it. It's a core platform-architecture portfolio piece.

**Brief:** Adventure Works has five product teams using models from two regions, a provisioned (PTU) deployment and pay-as-you-go deployments. Finance wants monthly chargeback per team, security wants no API keys in apps, and the platform team wants spillover from provisioned to standard capacity when PTUs are saturated.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Single gateway endpoint for all teams; apps authenticate with Entra ID (managed identity), never model keys |
| R2 | Gateway authenticates to back ends with its own managed identity |
| R3 | Back-end pool: provisioned deployment (priority 1) and standard deployments in two regions (priority 2), with circuit breaker on 429 and 5xx |
| R4 | Token-per-minute limits and monthly token quotas per team (subscription or product) |
| R5 | Emit token metrics with dimensions (team, model, API) for chargeback |
| R6 | Optional semantic caching for repeated prompts |
| R7 | Content safety and prompt logging policy appropriate to data classification |
| R8 | Private networking: gateway in VNet; back ends via private endpoints |
| R9 | Monthly chargeback report per team |
| R10 | Expose an internal MCP server through the gateway (stretch goal; see capstone 07) |

### Skills practised

Topics [11](../11-integration-messaging-and-apis/README.md), [15](../15-foundry-models-and-azure-openai/README.md), [29](../29-cost-management-and-finops/README.md), [26](../26-monitoring-and-observability/README.md), [34 file 05](../34-ai-solution-architecture/05-generative-ai-gateway-architecture.md) and [34 file 07](../34-ai-solution-architecture/07-multi-region-and-high-availability-ai.md).

## How it works on Azure

### Architecture

```text
Team apps (managed identity) ─▶ API Management (internal VNet / private endpoint)
   policies: validate-azure-ad-token → llm-token-limit (per subscription) → semantic cache lookup
             → set-backend-service (load-balanced pool with priorities + circuit breaker)
             → authentication-managed-identity → llm-emit-token-metric (team, model dimensions)
   ├─▶ Foundry deployment A (PTU, region 1)           priority 1
   ├─▶ Foundry deployment B (standard, region 1)      priority 2
   └─▶ Foundry deployment C (standard, region 2)      priority 2
Metrics → Application Insights / Azure Monitor → chargeback workbook / Power BI
```

### Policy fragment (illustrative; check policy reference for current attributes)

```xml
<inbound>
  <base />
  <validate-azure-ad-token tenant-id="{{tenant-id}}">
    <audiences><audience>api://ai-gateway</audience></audiences>
  </validate-azure-ad-token>
  <llm-token-limit counter-key="@(context.Subscription.Id)"
                   tokens-per-minute="20000" token-quota="5000000" token-quota-period="Monthly"
                   estimate-prompt-tokens="true" />
  <set-backend-service backend-id="foundry-pool" />
  <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
  <llm-emit-token-metric namespace="ai-gateway">
    <dimension name="Team" value="@(context.Subscription.Name)" />
    <dimension name="API ID" />
  </llm-emit-token-metric>
</inbound>
```

The **AI-Gateway** samples repository has working labs for load balancing, token limits, metrics, semantic caching and MCP. Use it to accelerate, then harden.

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Gateway design ADR (APIM tier, network mode, back-end strategy, quota model) |
| M2 | APIM with managed identity; import the model API; Entra token validation |
| M3 | Back-end pool with priorities and circuit breaker; failover test under load |
| M4 | Token limits and quotas per team product or subscription |
| M5 | Token metrics, dashboard and chargeback report |
| M6 | Semantic caching experiment (hit rate, latency, quality risk) |
| M7 | Private networking and IaC for everything |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| APIM tier | Check current tier support for the AI gateway policies, VNet integration and scale (Premium or v2 tiers); balance cost against network needs |
| Quota key | Subscription per team is simple; JWT claims enable per-app or per-user limits |
| Spillover | Provisioned first, then standard; Foundry also offers native spillover for provisioned deployments, so compare it with gateway-level routing |
| Semantic cache | Only for repetitive, non-personalised prompts; risk of stale or cross-user answers |
| Logging prompts | Log metadata by default; log content only where policy allows and data is protected |

## Hands-on

1. Complete milestones M1–M7.
2. Load test with one team exceeding its limit and show 429s for that team only.
3. Saturate the PTU deployment and show traffic moving to standard back ends.
4. Produce a monthly chargeback table from metrics.
5. Write a runbook for adding a new team.

## Security, cost and reliability notes

- No model keys anywhere: Entra for clients, managed identity for back ends; disable key access on back ends where possible.
- APIM becomes a critical dependency: use zone redundancy and multi-region where needed, and include it in DR.
- Gateway cost is small compared with model cost, but Premium tiers are significant; size accordingly.

## Common mistakes

| Mistake | Fix |
|---|---|
| Passing back-end keys to apps | Managed identity at the gateway |
| Retrying 429s on the same back end | Circuit breaker and priority pool |
| Quotas without metrics | Emit token metrics with team dimensions |
| Caching personalised answers | Restrict caching scope |

## Check yourself

1. Why use a gateway in front of Foundry deployments?
2. How do priorities and circuit breakers implement spillover?
3. How does chargeback work?
4. When is semantic caching inappropriate?
5. How do apps authenticate without keys?

<details>
<summary>Answers</summary>

1. For per-team quotas, load balancing and failover, consistent security, usage metrics and chargeback, none of which Foundry provides across deployments by itself.
2. Requests go to priority 1 (PTU) first; on 429 or errors the circuit breaker trips and traffic flows to priority 2 back ends.
3. Token metrics are emitted with team dimensions, aggregated monthly and priced per model.
4. For personalised, permission-sensitive or rapidly changing answers.
5. Apps get Entra tokens via managed identity; the gateway validates them and calls back ends with its own managed identity.

</details>

## References

- [AI gateway capabilities in API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [llm-token-limit policy](https://learn.microsoft.com/azure/api-management/llm-token-limit-policy)
- [llm-emit-token-metric policy](https://learn.microsoft.com/azure/api-management/llm-emit-token-metric-policy)
- [Use a gateway in front of multiple Azure OpenAI deployments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-guide)
- [AI-Gateway samples](https://github.com/Azure-Samples/AI-Gateway)
