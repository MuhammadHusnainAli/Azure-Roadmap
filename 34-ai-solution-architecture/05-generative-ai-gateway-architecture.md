# Generative AI Gateway Architecture

## Why it matters

When ten teams build AI apps, each connecting directly to model deployments with its own keys, the organisation loses control: no consistent authentication, no per-team quotas, no central logging of usage, no failover when one deployment throttles, no chargeback, and no single place to apply policies. A **generative AI gateway** sits between consumers and models (and increasingly agents and MCP tools), solving reliability, security, cost and operational challenges in one layer. Microsoft's Architecture Center guidance and **Azure API Management**'s GenAI gateway capabilities are the standard implementation. [Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md) covers APIM features; this file covers the architecture decisions.

## Key concepts

### Challenges a gateway solves

| Pillar | Challenge | Gateway capability |
|---|---|---|
| **Reliability** | Throttling (429), regional outages, capacity limits | Load-balanced backend pools, priority/weights, circuit breakers honouring `Retry-After`, failover across deployments/regions |
| **Security** | Keys spread across apps, no central auth, unfiltered content | Entra authentication for consumers, managed identity to backends (no keys), network isolation, content safety integration |
| **Cost optimisation** | No per-consumer visibility or limits | Token limits/quotas per consumer, token metrics with dimensions, semantic caching, routing to cheaper models |
| **Operational excellence** | Inconsistent logging, hard to change models | Central logging of prompts/usage (governed), versioning, abstraction of backend deployments |
| **Performance efficiency** | PTU underused while PAYG throttles | Route baseline to PTU, spill to PAYG; latency-based routing |

### APIM GenAI capabilities

| Capability | Policy/feature |
|---|---|
| Token rate limiting/quota | `llm-token-limit` / `azure-openai-token-limit` |
| Token usage metrics | `llm-emit-token-metric` with custom dimensions (consumer, app, team) |
| Semantic caching | `llm-semantic-cache-lookup` / `-store` with Azure Managed Redis and embeddings |
| Backend pools and circuit breakers | Load-balanced pools (round-robin, weighted, priority), circuit breaker rules |
| Content safety | `llm-content-safety` policy integrating Azure AI Content Safety |
| Logging | Prompt/completion logging to Application Insights/Log Analytics (with governance) |
| Managed identity auth | `authentication-managed-identity` to Foundry/Azure OpenAI |
| MCP and A2A | Expose and govern MCP servers and agent APIs; register in API Center |
| Foundry integration | AI gateway linked to Foundry for Control Plane governance (Topic 14 file 09) |

### Topologies

| Topology | Description | Fit |
|---|---|---|
| **Central gateway** | One platform-owned gateway for all AI consumers | Strong governance, shared PTUs, central chargeback |
| **Federated gateways** | Gateways per business unit/region with common policy templates | Large or regulated organisations, data residency boundaries |
| **Workload gateway** | Gateway owned by one workload | Single application needing failover and limits |
| **Hybrid** | Central gateway for shared models; workload gateways for specific needs | Common in practice |

### Multi-backend routing

```text
Consumer → APIM (auth, token limits, metrics, cache)
         → Backend pool "gpt-chat":
             priority 1: PTU deployment (Region A)
             priority 2: Data Zone standard deployment (Region A)
             priority 3: Data Zone standard deployment (Region B)   [if residency allows]
         → circuit breaker per backend (429/5xx, honour Retry-After)
```

### Chargeback

Emit token metrics with dimensions (subscription key/app ID/team), combine with effective price per model (including PTU amortisation) and publish monthly showback (Topic 29 file 04).

## How it works on Azure

APIM policy fragment for a chat API:

```xml
<inbound>
  <base />
  <validate-azure-ad-token tenant-id="{{tenant-id}}">
    <audiences><audience>api://ai-gateway</audience></audiences>
  </validate-azure-ad-token>
  <llm-token-limit counter-key="@(context.Request.Headers.GetValueOrDefault("x-app-id","unknown"))"
                   tokens-per-minute="50000" estimate-prompt-tokens="true" remaining-tokens-header-name="x-remaining-tokens" />
  <llm-emit-token-metric namespace="ai-gateway">
    <dimension name="App" value="@(context.Request.Headers.GetValueOrDefault("x-app-id","unknown"))" />
    <dimension name="Deployment" value="@(context.Request.MatchedParameters.GetValueOrDefault("deployment-id",""))" />
  </llm-emit-token-metric>
  <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
  <set-backend-service backend-id="gpt-chat-pool" />
</inbound>
```

(Policy names and attributes follow APIM's GenAI gateway documentation; verify current syntax for your API version.)

Agents can also consume gateway-fronted models as custom API tools, and the Foundry Control Plane can register custom agents and MCP tools through the AI gateway (Topic 23 file 10).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Central vs federated** | Central for consistency and shared capacity; federated where residency, autonomy or scale demand. |
| **Logging prompts** | Log metadata always; log content only with privacy approval, redaction and retention controls. |
| **Semantic caching** | Effective for repeated questions; risk of stale or cross-user leakage; scope caches by tenant/permissions. |
| **Gateway SKU** | Premium/v2 tiers for VNet, multi-region and scale; self-hosted gateways for edge. |
| **Latency** | Gateway adds milliseconds; colocate with backends. |
| **Agent traffic** | Govern MCP tools and agent APIs through the same gateway for consistent security. |

## Hands-on

1. Deploy APIM with a backend pool of two model deployments and circuit breakers; load test to trigger failover.
2. Add token limits and token metrics per app; build a chargeback query.
3. Enable semantic caching with Azure Managed Redis; measure hit rate and cost savings.
4. Replace application API keys with Entra auth to the gateway and managed identity to models.
5. Expose an MCP server through APIM and register it in API Center.

## Security, cost and reliability notes

- **Single point of failure**: deploy the gateway zone-redundantly and multi-region for critical workloads.
- **Prompt data in logs** is sensitive; govern access and retention.
- **Cost**: APIM tier plus Redis for caching; offset by better PTU utilisation and caching savings.
- **Cache safety**: don't serve one user's permission-specific answer to another.

## Common mistakes

| Mistake | Fix |
|---|---|
| Each app holding model keys | Gateway with Entra and managed identity |
| No per-consumer limits | Token limits and quotas |
| Gateway in one zone only | Zone redundancy and multi-region |
| Logging full prompts without governance | Redaction, access control, retention |
| Global semantic cache across tenants | Scope caches |

## Check yourself

1. Which challenges does a GenAI gateway address per pillar?
2. Name five APIM GenAI capabilities.
3. Compare central and federated gateway topologies.
4. Describe a multi-backend routing setup.
5. How is chargeback implemented?
6. What are the risks of semantic caching?

<details>
<summary>Answers</summary>

1. Reliability (throttling, failover), security (central auth, no keys, content safety), cost (limits, metrics, caching), operations (logging, abstraction), performance (PTU utilisation, routing).
2. Any five: token limits, token metrics, semantic caching, backend pools with circuit breakers, content safety policy, logging, managed identity auth, MCP/A2A exposure, Foundry integration.
3. Central: one platform gateway with strong governance and shared capacity; federated: multiple gateways with common templates for autonomy, residency or scale.
4. Priority-based pool: PTU first, then standard deployments in the same and other regions, with circuit breakers honouring Retry-After.
5. Emit token metrics with consumer dimensions and combine with effective model prices for showback/chargeback.
6. Stale answers and leaking permission-specific content across users or tenants if caches aren't scoped.

</details>

## References

- [Access models through a generative AI gateway](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-guide)
- [Gateway in front of multiple deployments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
- [API Management GenAI gateway capabilities](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [LLM emit token metric policy](https://learn.microsoft.com/azure/api-management/llm-emit-token-metric-policy)
- [Semantic cache lookup policy](https://learn.microsoft.com/azure/api-management/azure-openai-semantic-cache-lookup-policy)
