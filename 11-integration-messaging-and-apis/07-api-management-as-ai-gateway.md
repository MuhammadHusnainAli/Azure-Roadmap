# API Management as AI Gateway

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Once more than one application calls a model, a few problems appear together. One app uses up the whole tokens-per-minute (TPM) quota, keys get shared, nobody knows which team spent what, a regional deployment throttles with 429s, and agents start calling tools (MCP servers) that nobody governs. The **AI gateway** in Azure API Management deals with these centrally. It applies token-based rate limits, load balancing and circuit breaking across model deployments, semantic caching, token metrics, content safety, managed-identity authentication, and the ability to expose and govern **MCP servers** and **A2A agent APIs**. It's the standard "AI hub gateway" pattern in Microsoft reference architectures.

## Key concepts

### What the AI gateway is

It isn't a separate product. It's a set of API Management capabilities (policies, backends, import wizards and monitoring) aimed at AI back ends. It can front:

| Back end | Notes |
|---|---|
| **Language model APIs** | OpenAI Chat Completions or Responses API schema (Azure OpenAI and other Foundry Models); **Anthropic Messages API** (v2 tiers); Google Vertex AI API; models hosted elsewhere, for example Amazon Bedrock |
| **Unified model API** (Preview) | One OpenAI-compatible endpoint in front of several providers, with automatic format translation |
| **Remote MCP servers** | Expose a REST API as an MCP server, or proxy an existing MCP server |
| **A2A agent APIs** | Import and govern Agent-to-Agent protocol APIs |
| **Self-hosted models** | Any HTTP endpoint you run (vLLM, managed compute, AKS) |

### Capability map

| Need | AI gateway feature |
|---|---|
| Stop one consumer exhausting TPM | `llm-token-limit` (TPM and/or token quota per counter key) |
| Know who used how many tokens | `llm-emit-token-metric` with custom dimensions; LLM logging and built-in dashboard |
| Spread load and use PTU first | **Backend pools** with priority and weight |
| Back off from throttled or failing deployments | **Circuit breaker** on backends, honouring `Retry-After` |
| Keep chat sessions on one back end | **Session awareness** (cookie-based affinity) |
| Reduce repeat calls | `llm-semantic-cache-lookup` / `llm-semantic-cache-store` with Azure Managed Redis |
| No API keys to the model | Managed identity from APIM to Foundry |
| Moderate prompts | `llm-content-safety` (Azure AI Content Safety, including Prompt Shields) |
| Govern agent tools | MCP server exposure with JWT validation, rate limits, IP filtering, logging |

The older `azure-openai-*` policies (`azure-openai-token-limit`, `azure-openai-emit-token-metric`, `azure-openai-semantic-cache-*`) still exist for Azure OpenAI-specific APIs. The `llm-*` equivalents work across supported model APIs and are the ones to learn now.

## How it works on Azure

### Reference architecture

```text
Apps / agents ──(Entra token + subscription key)──► APIM AI gateway
                                                   │ validate-azure-ad-token
                                                   │ llm-token-limit (per app)
                                                   │ llm-semantic-cache-lookup
                                                   │ llm-content-safety
                                                   │ set-backend-service → pool
                                                   ▼
                       ┌──────── pool "chat" ────────┐
                       │ priority 1: PTU deployment   │  (circuit breaker on 429)
                       │ priority 2: Global Standard  │  weight 50 / region A
                       │ priority 2: Global Standard  │  weight 50 / region B
                       └──────────────────────────────┘
                                                   │ llm-emit-token-metric → App Insights
                                                   │ logs (prompts, completions, tokens) → Log Analytics
```

### 1. Import a Foundry model API

In the portal: **APIs → Add API → Microsoft Foundry** (or **Language model API** for OpenAI-compatible endpoints). The wizard creates the API and a backend, configures **managed identity** authentication (`https://cognitiveservices.azure.com`), and can pre-add token limit and token metric policies. Grant the APIM identity the **Cognitive Services User** role (or **Cognitive Services OpenAI User** for Azure OpenAI only) on the Foundry resource.

### 2. Token limits per consumer

```xml
<inbound>
  <base />
  <!-- 20k TPM and 5M tokens a month per subscription (one subscription per app) -->
  <llm-token-limit counter-key="@(context.Subscription.Id)"
                   tokens-per-minute="20000"
                   token-quota="5000000" token-quota-period="Monthly"
                   estimate-prompt-tokens="false"
                   remaining-tokens-header-name="x-remaining-tokens"
                   tokens-consumed-header-name="x-tokens-consumed" />
</inbound>
```

Behaviour to know (checked 2026-11):

| Point | Detail |
|---|---|
| Responses | TPM exceeded → **429**; quota exceeded → **403** |
| `estimate-prompt-tokens` | `true` estimates before sending (can block early, costs some performance); `false` uses actual usage from the response |
| Streaming | With `stream: true`, prompt and completion tokens are always estimated |
| Scope | Counters are per gateway (regional gateways and workspace gateways count separately) |
| v2 tiers | Use a token bucket algorithm; keep `tokens-per-minute` consistent when the same key is used at several scopes |
| Tiers | All tiers except Consumption |

### 3. Load balancing and circuit breaking

Define each model deployment as a backend with a circuit breaker, then group them in a pool:

```bicep
resource ptu 'Microsoft.ApiManagement/service/backends@2024-05-01' = {
  parent: apim
  name: 'aoai-ptu-swc'
  properties: {
    url: 'https://foundry-swc.openai.azure.com/openai'
    protocol: 'http'
    // Authenticate with managed identity: configure it on the backend (portal: Authorization credentials →
    // Managed identity, resource https://cognitiveservices.azure.com) or use the
    // <authentication-managed-identity resource="https://cognitiveservices.azure.com" /> policy.
    circuitBreaker: {
      rules: [ {
        name: 'throttle'
        failureCondition: {
          count: 1
          interval: 'PT10S'
          statusCodeRanges: [ { min: 429, max: 429 }, { min: 500, max: 599 } ]
        }
        tripDuration: 'PT10S'
        acceptRetryAfter: true      // trip for as long as the back end's Retry-After says
      } ]
    }
  }
}

resource pool 'Microsoft.ApiManagement/service/backends@2024-05-01' = {
  parent: apim
  name: 'chat-pool'
  properties: {
    type: 'Pool'
    pool: {
      services: [
        { id: ptu.id,       priority: 1 }
        { id: paygoA.id,    priority: 2, weight: 50 }
        { id: paygoB.id,    priority: 2, weight: 50 }
      ]
    }
  }
}
```

```xml
<set-backend-service backend-id="chat-pool" />
<!-- Optional: retry on the next backend when one throttles -->
```

Rules (checked 2026-11): up to **30 backends per pool**; **one** circuit breaker rule per backend; lower-priority groups receive traffic only when all higher-priority backends are tripped; load balancing and breaker state are **approximate** because gateway instances don't synchronise; the circuit breaker isn't supported in the **Consumption** tier. Azure OpenAI can return very long `Retry-After` values, which is why `acceptRetryAfter` matters.

### 4. Semantic caching

```xml
<inbound>
  <base />
  <llm-semantic-cache-lookup score-threshold="0.05"
        embeddings-backend-id="embeddings-backend"
        embeddings-backend-auth="system-assigned"
        ignore-system-messages="true">
    <vary-by>@(context.Subscription.Id)</vary-by>   <!-- don't share answers across tenants -->
  </llm-semantic-cache-lookup>
  <rate-limit calls="60" renewal-period="60" />      <!-- protects the back end if the cache is unavailable -->
</inbound>
<outbound>
  <llm-semantic-cache-store duration="300" />
  <base />
</outbound>
```

| Point | Detail |
|---|---|
| Requirements | An **external cache** compatible with RediSearch (Azure Managed Redis with the RediSearch module) and an **embeddings** model backend |
| `score-threshold` | 0.0–1.0, and it is a **distance**: **lower values require closer matches**. Start around 0.05; above about 0.2 risks wrong hits |
| Isolation | Always partition with `vary-by` (user, tenant, subscription) when answers may be user-specific |
| Risk | Similar ≠ identical. Cached answers can be wrong, stale or unsafe for the new request, so evaluate before enabling |

### 5. Observability

```xml
<llm-emit-token-metric namespace="ai-gateway">
  <dimension name="Subscription ID" />
  <dimension name="API ID" />
  <dimension name="Team" value="@(context.Request.Headers.GetValueOrDefault("x-team","unknown"))" />
</llm-emit-token-metric>
```

Emits prompt, completion and total token metrics to **Application Insights** (custom metrics must be enabled). Enable **LLM logging** in diagnostic settings to send token usage, prompts and completions to Log Analytics, and use the built-in **language model analytics** dashboard for chargeback by consumer. Log prompts and completions only when your data-handling policy allows it.

### 6. Governing MCP servers

| Option | What happens |
|---|---|
| **Expose a REST API as an MCP server** | Choose operations of an API already in APIM; each becomes an MCP **tool**. APIM hosts the MCP endpoint (Streamable HTTP) |
| **Expose an existing MCP server** | APIM proxies a remote MCP server (for example one on Functions, Container Apps or Logic Apps) and applies policies |

Availability and limits (checked 2026-11):

- Tiers: Developer, Basic, Standard, Premium, Basic v2, Standard v2, Premium v2 (and the self-hosted gateway). **Not** supported in workspaces.
- APIM supports MCP **tools** only, not MCP resources or prompts.
- Policies apply to all tools on the server: `validate-azure-ad-token` / `validate-jwt`, `rate-limit-by-key`, `ip-filter`, caching, and credential manager for OAuth to downstream APIs.
- Register the servers in **API Center** (file 08) so developers and agents can discover them.

### 7. AI gateway in Microsoft Foundry (Preview)

A Foundry resource can be associated with an APIM instance (new or existing) as its **AI gateway**. You can then set token limits for model deployments from the Foundry portal, register agents running anywhere into the **Foundry Control Plane**, and govern MCP tools. For custom policies and enterprise networking, you still use the full APIM experience. See topic 14.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Tier | Standard v2 for most AI gateways; Premium (v2) for full network isolation and zones; classic Premium for multi-region gateways. Consumption lacks circuit breakers and several AI features |
| Counter key | Subscription per application is simplest. Use a JWT claim (app ID, user, team) for finer control |
| PTU first, PAYG spill | Priority pools with PTU at priority 1 and Global Standard at priority 2. Foundry also has native PTU **spillover** (topic 15); choose one owner of the behaviour |
| Retry in gateway vs client | Gateway retry to another backend is transparent to clients but adds latency; keep client retries short when the gateway retries |
| Semantic cache | Good for FAQ-style, non-personalised traffic. Avoid for personalised, time-sensitive or high-risk answers |
| Prompt logging | Valuable for evaluation and audit; a data-protection risk. Decide retention, masking and access deliberately |
| Expose MCP via APIM vs directly | APIM when tools must be centrally authenticated, rate-limited, observed and catalogued; direct for internal prototypes |

## Hands-on

Use the Azure-Samples **AI-Gateway** labs (GitHub `Azure-Samples/AI-Gateway`). Each lab deploys with Bicep and runs from a notebook.

1. Deploy a Standard v2 APIM and two Azure OpenAI deployments in different regions. Import them with the Foundry wizard using managed identity.
2. Add `llm-token-limit` at 1,000 TPM and send a burst; observe 429 and `x-remaining-tokens`.
3. Build a priority pool with a circuit breaker. Lower the first deployment's TPM to force throttling and watch traffic shift.
4. Enable `llm-emit-token-metric` and query the custom metric in Application Insights by subscription.
5. Add Azure Managed Redis with RediSearch and enable semantic caching; ask paraphrased questions and compare latency.
6. Expose an existing REST API as an MCP server and call it from GitHub Copilot in VS Code with an Entra token.

## Security, cost and reliability notes

- **Security:** clients authenticate to APIM with Entra ID tokens; APIM authenticates to Foundry with **managed identity**. Disable key auth on the Foundry resource so the gateway can't be bypassed.
- **Security:** put the Foundry resource behind a **private endpoint** and allow only APIM to reach it.
- **Security:** `embeddings-backend-id` doesn't check linked access, so anyone who can edit policies can route calls through any backend using APIM's identity. Restrict who can author policies.
- **Security:** add `llm-content-safety` for prompt-attack detection on public-facing APIs.
- **Cost:** token metrics per consumer are the basis for chargeback and budgets. Semantic caching and token quotas directly cut spend.
- **Reliability:** combine multi-region back ends, priority pools and circuit breakers. Remember the gateway itself must be resilient (zones, or two instances behind Front Door).

## Common mistakes

- Treating `score-threshold` as a similarity score where higher means stricter. It's the opposite.
- One shared subscription key for all apps, so token limits and metrics can't separate them.
- Leaving key authentication enabled on the Foundry resource, so clients bypass the gateway.
- No `vary-by` on semantic cache in a multi-tenant app, which leaks answers across users.
- Using `azure-openai-*` policies for non-Azure OpenAI models.
- Exposing MCP servers without token validation because "it's internal".

## Check yourself

1. What status codes does `llm-token-limit` return for TPM and quota violations?
2. How would you make sure PTU capacity is used before pay-as-you-go?
3. What does `acceptRetryAfter` do in a circuit breaker rule?
4. What two resources does semantic caching require?
5. Does a higher `score-threshold` make caching stricter or looser?
6. Name two ways APIM can expose MCP servers, and one limitation.
7. How do you attribute token usage to teams?
8. Why should key authentication be disabled on the model resource behind the gateway?

<details>
<summary>Answers</summary>

1. 429 Too Many Requests when the TPM rate is exceeded; 403 Forbidden when the token quota is exceeded.
2. Put the PTU deployment in a backend pool at priority 1 and pay-as-you-go deployments at priority 2, with circuit breakers so traffic moves when PTU throttles.
3. It makes the breaker stay open for the duration given in the back end's `Retry-After` header, instead of a fixed trip duration.
4. A RediSearch-compatible external cache (for example Azure Managed Redis) and an embeddings model backend.
5. Looser: the threshold is a distance, so lower values demand closer matches.
6. Expose a REST API managed in APIM as an MCP server, or proxy an existing MCP server. Limitations: tools only (no resources or prompts), and not supported in workspaces.
7. Use `llm-emit-token-metric` with dimensions such as subscription ID, API ID or a team header or claim, and analyse in Application Insights; or use LLM logs and the analytics dashboard.
8. So that every call must go through the gateway's policies (limits, logging, safety) and authenticate with Entra ID, instead of bypassing it with a key.

</details>

## References

- [AI gateway capabilities in API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [llm-token-limit policy](https://learn.microsoft.com/azure/api-management/llm-token-limit-policy)
- [llm-emit-token-metric policy](https://learn.microsoft.com/azure/api-management/llm-emit-token-metric-policy)
- [llm-semantic-cache-lookup policy](https://learn.microsoft.com/azure/api-management/llm-semantic-cache-lookup-policy)
- [Enable semantic caching for LLM APIs](https://learn.microsoft.com/azure/api-management/azure-openai-enable-semantic-caching)
- [Backends: pools and circuit breaker](https://learn.microsoft.com/azure/api-management/backends)
- [MCP servers in API Management](https://learn.microsoft.com/azure/api-management/mcp-server-overview)
- [Expose a REST API as an MCP server](https://learn.microsoft.com/azure/api-management/export-rest-mcp-server)
- [Use a gateway in front of multiple Azure OpenAI deployments (Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
- [AI Gateway labs (GitHub)](https://github.com/Azure-Samples/AI-Gateway)
