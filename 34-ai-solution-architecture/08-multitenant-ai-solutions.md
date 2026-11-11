# Multitenant AI Solutions

## Why it matters

SaaS providers are adding AI to their products, and internal platform teams offer shared AI services to many business units. Multitenant AI introduces isolation risks traditional SaaS never had: one tenant's documents retrieved into another tenant's answer, agent memory leaking across customers, a semantic cache returning another tenant's response, fine-tuned models trained on mixed customer data, prompt injection in one tenant's content affecting shared agents, and one heavy tenant exhausting shared model capacity. This file applies multitenant architecture principles ([Topic 33 file 06](../33-solution-architecture/06-multitenant-architecture.md)) to models, indexes, memory and agents, based on the Architecture Center's secure multitenant RAG and multitenant AI guidance.

## Key concepts

### Isolation by AI component

| Component | Isolation options | Notes |
|---|---|---|
| **Models (shared base models)** | Shared deployments with per-tenant quotas; dedicated deployments/PTUs for premium tenants | Base models don't retain tenant data between calls; isolation is about capacity and configuration |
| **Fine-tuned models** | Per-tenant fine-tunes; never train shared models on tenant data without consent | Contractual and privacy implications |
| **Grounding data stores (search indexes)** | **Store per tenant** (index or service per tenant), **multitenant store with tenant filters**, or **shared stores** for common content | Choose per data sensitivity and scale; service limits matter (indexes per service) |
| **Conversation and agent state** | Tenant-scoped conversations; ownership checks; per-tenant Cosmos DB containers/partitions or accounts | Agent Service doesn't enforce per-user authorisation on conversations (file 02) |
| **Agent memory** | Memory scoped by tenant and user (scope parameter); no cross-tenant memory stores | Review what memory extracts and retains |
| **Agents** | Shared agent definitions with tenant context; per-tenant agents for customised behaviour | Tenant-specific instructions/tools increase management overhead |
| **Tools and APIs** | Tools call back-end APIs with tenant context derived from tokens; on-behalf-of where possible | Never let the model choose the tenant ID |
| **Caches** | Semantic cache keys include tenant (and permission scope) | Prevent cross-tenant cache hits |
| **Telemetry** | Tenant ID in traces; content recording governed per tenant contracts | Supports metering and investigations |
| **Encryption** | Per-tenant CMK for premium tenants where supported | Crypto isolation |

### Store isolation models (secure multitenant RAG guidance)

| Model | Description | Pros | Cons |
|---|---|---|---|
| **Store per tenant** | Separate index/database (or service) per tenant | Strong isolation, easy offboarding, per-tenant scaling | Cost, service limits, management overhead |
| **Multitenant store** | Shared index with tenant ID field and mandatory filters | Efficient for many small tenants | Filter must never be bypassed; noisy neighbour |
| **Shared store** | Content common to all tenants (product docs) | Simple | Only for non-tenant-specific data |

Data filtering must be enforced by the application/orchestrator (not the model), using tenant and user identity from validated tokens.

### Identity and authorisation

Define tenant and authorisation requirements: tenant users authenticate with Entra (multitenant app or External ID); the orchestrator resolves tenant ID from claims, applies tenant filters and user-level permissions (security trimming within the tenant), and uses tenant-scoped credentials for downstream calls.

### Noisy neighbours and fairness

Per-tenant token limits in the AI gateway, separate PTU pools for premium tiers, queueing for batch workloads, and per-tenant cost metering for pricing.

## How it works on Azure

Reference architecture for a B2B SaaS assistant:

```text
Tenant users → Front Door (custom domains) → App (tenant resolution from token)
→ Orchestrator / Foundry agent (shared definition, tenant context injected server-side)
   tools: search_knowledge(tenant_id from context, user groups) → AI Search
          - premium tenants: dedicated index per tenant
          - standard tenants: shared index with mandatory filter tenant_id eq '<tid>'
          get_orders(tenant_id, user OBO) → tenant API
→ AI gateway: per-tenant token limits, metrics (tenant dimension), cache keyed by tenant
→ State: Cosmos DB with partition key /tenantId; conversation ownership checks
→ Metering: tokens, searches, storage per tenant → billing
```

Tool implementation enforcing tenant context (sketch):

```python
def search_knowledge(query: str, ctx: RequestContext):
    # ctx comes from the validated token on the server, never from the model's arguments
    filt = f"tenant_id eq '{ctx.tenant_id}' and group_ids/any(g: search.in(g, '{','.join(ctx.group_ids)}'))"
    index = premium_index_for(ctx.tenant_id) or "kb-shared-tenants"
    return search_client(index).search(search_text=query, filter=filt, top=8, query_type="semantic")
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Default model** | Shared models with quotas; multitenant indexes with strict filters for small tenants; dedicated indexes/deployments for premium/regulated tenants. |
| **Tenant context injection** | Server-side only; tools ignore tenant parameters from the model. |
| **Fine-tuning** | Per tenant only with explicit consent; prefer RAG for tenant customisation. |
| **Memory** | Scope to tenant and user; allow tenants to opt out and delete. |
| **Evaluation** | Include cross-tenant leakage tests and tenant-specific evaluation sets. |
| **Commercials** | Pricing tiers aligned with isolation level and capacity guarantees. |

## Hands-on

1. Choose isolation models for 500 small tenants and 5 regulated enterprise tenants.
2. Implement a search tool that injects tenant filters from server-side context and test that model-supplied tenant IDs are ignored.
3. Configure per-tenant token limits and metrics in APIM.
4. Write automated cross-tenant leakage tests (retrieval, cache, conversation access).
5. Build a per-tenant cost report.

## Security, cost and reliability notes

- **Cross-tenant leakage** is the critical risk; test continuously, including red teaming with prompt injection attempting tenant escape.
- **Contracts and privacy**: document data use, retention and model training policies per tenant.
- **Cost**: dedicated resources per tenant must be priced in; shared resources need fairness controls.
- **Blast radius**: stamps per tenant group for large SaaS (Topic 33 file 07).

## Common mistakes

| Mistake | Fix |
|---|---|
| Letting the model pass tenant IDs to tools | Server-side context |
| Filters optional in code paths | Mandatory filter enforcement layer |
| Global semantic cache | Tenant-scoped cache keys |
| Training shared models on tenant data | Per-tenant fine-tunes with consent, or RAG |
| No per-tenant quotas | Gateway limits |

## Check yourself

1. Which AI components need tenant isolation, and how?
2. Compare store-per-tenant, multitenant and shared stores.
3. Why must tenant filtering be enforced outside the model?
4. How do you prevent semantic cache leakage?
5. How should fine-tuning be handled in multitenant SaaS?
6. What tests should a multitenant AI solution include?

<details>
<summary>Answers</summary>

1. Models (quotas/dedicated), fine-tunes (per tenant), indexes (per-tenant or filtered), conversations/state (scoped, ownership checks), memory (tenant/user scope), agents and tools (server-side context), caches (tenant keys), telemetry (tenant IDs), encryption (per-tenant keys).
2. Store per tenant: strongest isolation, higher cost/limits; multitenant store: efficient with mandatory filters; shared store: only for non-tenant data.
3. Models can be manipulated (prompt injection) or make mistakes; isolation must be deterministic in code.
4. Include tenant and permission scope in cache keys.
5. Only per tenant with explicit consent; prefer RAG for customisation.
6. Cross-tenant leakage tests for retrieval, cache, conversations and memory; prompt injection red teaming; tenant-specific evaluation.

</details>

## References

- [Design a secure multitenant RAG inferencing solution](https://learn.microsoft.com/azure/architecture/ai-ml/guide/secure-multitenant-rag)
- [Multitenancy and AI/ML](https://learn.microsoft.com/azure/architecture/guide/multitenant/approaches/ai-machine-learning)
- [Multitenancy and Azure OpenAI](https://learn.microsoft.com/azure/architecture/guide/multitenant/service/openai)
- [Multitenant architecture overview](https://learn.microsoft.com/azure/architecture/guide/multitenant/overview)
