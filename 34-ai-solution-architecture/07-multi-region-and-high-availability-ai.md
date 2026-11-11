# Multi-Region and High-Availability AI

## Why it matters

AI features are becoming core parts of customer journeys and internal operations, so their availability matters as much as the rest of the application. Yet AI has unusual availability constraints: model capacity is quota-bound and varies by region, standard deployments are single-region without zones, PTUs are reserved in specific regions, data residency limits where prompts can be processed, retrieval indexes and agent state live in regional services, and Foundry provides no automatic failover. This file designs high availability and multi-region resilience for AI workloads, combining capacity strategy, gateway failover and data-zone constraints. It extends [Topic 28 file 06](../28-resiliency-backup-and-disaster-recovery/06-resilience-for-ai-workloads.md) with architecture patterns.

## Key concepts

### Availability building blocks

| Layer | In-region HA | Cross-region |
|---|---|---|
| **App/orchestrator** | Zone-redundant App Service/Container Apps/AKS | Active-active or warm standby stamps behind Front Door |
| **Models** | Multiple deployments; PTU + standard spillover; Data Zone/Global deployment types route across Microsoft capacity | Deployments in multiple regions behind an AI gateway |
| **Gateway** | Zone-redundant APIM | Multi-region APIM gateways (Premium) or per-region gateways behind Front Door |
| **Retrieval** | AI Search with 3+ replicas across zones | Second search service in another region, indexed from the same sources |
| **Agent state** | Zone-redundant Cosmos DB, ZRS Storage, zone-redundant Search | Cosmos DB multi-region; DR via redeploy and restore (Foundry has no automatic DR) |
| **Agent definitions** | As code | Redeploy to a secondary project |
| **Identity, secrets, DNS** | Zone-redundant | Available in both regions |

### Capacity strategy

| Strategy | Detail |
|---|---|
| **Baseline on PTU** | Predictable latency for steady load; reserve in primary region |
| **Spillover/burst on standard** | Data Zone or Global standard deployments absorb peaks |
| **Secondary region capacity** | Standard deployments with quota pre-approved; PTU in secondary only for strict RTO |
| **Model diversity** | Fallback to an alternative model (evaluated) when the primary model is constrained |
| **Quota management** | Request quotas in advance for DR regions; track per-subscription limits |

### Data residency constraints

| Deployment type | Processing scope | HA implication |
|---|---|---|
| **Global** | Any region with the model | Highest availability; may violate residency |
| **Data Zone** (EU, US) | Within the zone | Good availability within the zone; satisfies zone-level residency |
| **Regional standard/provisioned** | One region | Need multiple regional deployments + gateway for HA |

Choose the broadest deployment type your residency policy allows.

### Patterns

| Pattern | RTO | Cost | Notes |
|---|---|---|---|
| **Single region, zone-redundant, model spillover** | Minutes for zone/model issues; hours for regional outage | Low | Baseline for most |
| **Active-passive with warm standby** | Minutes–hours | Medium | Secondary project/agents deployed, indexes kept current, standard model capacity |
| **Active-active** | Near zero | High | Requires state strategy (shared Cosmos DB multi-region or stateless conversations) and dual indexing |
| **Degraded mode** | n/a | Low | When AI is unavailable, fall back to search-only, FAQ or human handoff |

## How it works on Azure

Active-passive AI chat across EU regions:

```text
Front Door (priority routing, health probes on composite /health incl. model and search checks)
├── Region A (primary): App (zone-redundant) → APIM (pool: PTU A → Data Zone EU standard)
│                       → Foundry project A (agents) → AI Search A (3 replicas) → Cosmos DB (multi-region, A write)
└── Region B (standby): App (scaled down) → APIM (pool: Data Zone EU standard B)
                        → Foundry project B (same agent definitions via pipeline) → AI Search B (dual indexing)
Failover: Front Door shifts traffic; Cosmos DB failover (or new conversations); runbook scales Region B
Degraded mode: if both model paths fail, UI offers search results and human handoff
```

Composite health check includes a cheap model call (or gateway health), a search query and a state store read, cached for seconds to avoid load.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Do you need multi-region?** | Many AI features tolerate degraded mode; reserve multi-region for critical journeys. |
| **Conversation continuity** | Usually acceptable to start new conversations after failover; replicate state only when required. |
| **Index consistency** | Dual indexing from sources is simpler than replicating indexes. |
| **PTU in secondary** | Expensive; use standard capacity unless RTO and latency demand PTU. |
| **Residency** | Use Data Zone deployments for EU/US boundaries; regional for stricter requirements. |
| **Testing** | Simulate model throttling, region loss and search outage regularly (Topic 28 file 07). |

## Hands-on

1. Design capacity for 2,000 concurrent users: PTU baseline, spillover and secondary region quota.
2. Configure APIM backend pools across two regions and test failover under load.
3. Deploy agents to two projects from the same pipeline; dual-index a knowledge source.
4. Implement a degraded mode in the UI.
5. Run a failover drill and record RTO and user impact.

## Security, cost and reliability notes

- **Security parity** in both regions: guardrails, private networking, identities, policies.
- **Cost**: dual indexes, standby compute, secondary capacity; balance against business impact.
- **Quotas** are a reliability dependency; request and verify in advance.
- **Model versions** should match across regions to keep behaviour consistent.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming Foundry fails over automatically | Gateway and redeploy strategy |
| No quota in DR region | Pre-approved quota |
| Different model versions per region | Version pinning across regions |
| Ignoring residency when choosing Global | Data Zone/regional per policy |
| No degraded mode | Graceful fallback UX |

## Check yourself

1. Which AI components need HA design, and how?
2. Describe a capacity strategy combining PTU and standard deployments.
3. How do Global, Data Zone and regional deployments affect HA and residency?
4. Compare active-passive and active-active AI patterns.
5. What is degraded mode?
6. Why include model and search checks in health probes?

<details>
<summary>Answers</summary>

1. Orchestrator (zone-redundant/stamps), models (multiple deployments, gateway), gateway (zone/multi-region), retrieval (replicas, second service), agent state (zone-redundant, multi-region Cosmos DB), definitions (as code), identity/secrets/DNS.
2. PTU for baseline in primary, spillover to Data Zone/Global standard, standard quota in secondary region, evaluated fallback model.
3. Global: most available, broadest processing; Data Zone: available within EU/US zone; regional: single region, needs multiple deployments and a gateway.
4. Active-passive: standby secondary, minutes-to-hours RTO, medium cost; active-active: both serve traffic, near-zero RTO, high cost and state complexity.
5. Fallback behaviour when AI is unavailable, such as search-only results, FAQs or human handoff.
6. Because the app can be "up" while AI dependencies fail; routing should reflect real user-facing capability.

</details>

## References

- [Baseline Foundry chat: multi-region and DR considerations](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
- [High availability and resiliency for Foundry](https://learn.microsoft.com/azure/foundry/how-to/high-availability-resiliency)
- [Deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
- [Gateway in front of multiple deployments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
- [Reliability in Azure AI Search](https://learn.microsoft.com/azure/search/search-reliability)
