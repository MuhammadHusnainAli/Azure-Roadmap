# Resilience for AI Workloads

## Why it matters

AI applications have failure modes that classic DR plans don't cover: a model deployment is throttled or a region runs out of capacity; a model version is retired; the vector index lives in one region; agent conversation state sits in a Cosmos DB account nobody configured for backup; a gateway sends all traffic to one endpoint. Microsoft is explicit that **Foundry itself doesn't provide automatic failover or disaster recovery**, and that you own the durability of the stateful dependencies behind Foundry Agent Service in Standard mode. Designing AI resilience means multi-deployment and multi-region model access through a gateway, recoverable knowledge and agent state, agent definitions as code, and tested recovery procedures.

## Key concepts

### What can fail

| Component | Failure | Mitigation |
|---|---|---|
| **Model endpoint** | 429 throttling, capacity limits, regional outage, model retirement | Multiple deployments (PTU + standard spillover), Global/Data Zone deployments, multiple regions behind a gateway, pinned versions with upgrade plans |
| **AI gateway** | Single region failure | Multi-region API Management or Front Door in front of gateways |
| **Retrieval** (AI Search, vector DBs) | Region or service outage, corrupted index | Replicas for in-region HA; second service in another region; rebuildable indexes from source |
| **Agent state** (Standard mode: Cosmos DB, AI Search, Storage) | Region outage, deletion, corruption | Customer-managed durability: Cosmos DB continuous backup, zone redundancy, locks, warm standby project |
| **Agent definitions, tools, prompts** | Deleted or changed project | Definitions as code; automated redeploy |
| **Tools and MCP servers** | Downstream API outage | Timeouts, fallbacks, circuit breakers |
| **Foundry resource/project** | Accidental deletion | Resource locks, RBAC, IaC redeploy |

### Model access resilience

| Technique | Detail |
|---|---|
| **Deployment types** | **Global** and **Data Zone** deployments route across Microsoft's capacity automatically within their scope; regional deployments are tied to one region ([Topic 15 file 04](../15-foundry-models-and-azure-openai/04-deployment-types.md)) |
| **Spillover** | Provisioned (PTU) deployments spill excess traffic to a standard deployment |
| **Gateway backend pools** | API Management **load-balanced backend pools** (round-robin, weighted, priority) with **circuit breakers** that honour `Retry-After`; route 429/5xx to the next backend |
| **Client retries** | Exponential backoff respecting `retry-after`; idempotent requests |
| **Model portability** | Abstract model names; evaluate alternates so a fallback model is ready |

### Agent Service DR (Standard mode)

Microsoft's guidance:

- You and Microsoft jointly operate Agent Service: Microsoft runs the control plane and runtime; **you own durability of Cosmos DB, AI Search and Storage** (the capability host dependencies).
- The **project** is usually the recovery unit; project data can't be moved into another project.
- **DR readiness checklist**: choose a recovery strategy per project (for example warm standby, or reconstruction); configure baseline protections; **treat agent definitions as code**; automate redeployment including client updates for new agent IDs; run drills; monitor dependencies with alerts.
- Enable **Cosmos DB continuous backup** (7, 30 or 35 days, per the Foundry guidance) **before** an incident; point-in-time restore is unavailable otherwise.
- Use **resource locks** on critical resources (requires Owner/User Access Administrator to manage).
- Basic mode uses Microsoft-managed storage with very limited recovery options; use Standard mode for production.

## How it works on Azure

Multi-region model access through API Management:

```text
Clients ─► Front Door ─► APIM (multi-region gateways)
                         └─ backend pool "chat-models" (priority + weights, circuit breaker on 429/5xx):
                             1. PTU deployment, Sweden Central (priority 1)
                             2. Data Zone standard deployment, EU (priority 2)
                             3. Global standard deployment (priority 3, only if policy allows)
```

Bicep for a backend pool with circuit breakers (simplified):

```bicep
resource beSweden 'Microsoft.ApiManagement/service/backends@2024-05-01' = {
  parent: apim
  name: 'aoai-sweden-ptu'
  properties: {
    url: 'https://aif-swc.openai.azure.com/openai'
    protocol: 'http'
    circuitBreaker: { rules: [ {
      name: 'throttle'
      failureCondition: { count: 3, interval: 'PT10S', statusCodeRanges: [ { min: 429, max: 429 }, { min: 500, max: 599 } ] }
      tripDuration: 'PT10S'
      acceptRetryAfter: true
    } ] }
  }
}

resource pool 'Microsoft.ApiManagement/service/backends@2024-05-01' = {
  parent: apim
  name: 'chat-models'
  properties: {
    type: 'Pool'
    pool: { services: [
      { id: '/backends/${beSweden.name}', priority: 1 }
      { id: '/backends/aoai-eu-datazone', priority: 2 }
    ] }
  }
}
```

Agent warm standby: a second Foundry project (and resource) in the secondary region deployed from the same IaC, with its own Cosmos DB, Search and Storage; agent definitions deployed by the same pipeline (Topic 27 file 06); knowledge indexed in both; clients switch by configuration or gateway routing. Conversation history may not be preserved across regions unless you replicate it yourself, so decide whether conversations can restart.

Index recovery: keep source documents in GZRS storage and indexing pipelines in code so an index can be rebuilt; for large indexes, maintain a second search service in the secondary region.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Global vs regional deployments** | Global/Data Zone for availability and capacity when residency allows; regional plus multi-region gateway when it doesn't. |
| **PTU plus spillover** | Predictable latency with burst capacity. |
| **Conversation state DR** | Decide if users can restart conversations after failover; replicate only when necessary. |
| **Reconstruction vs warm standby** | Reconstruction (redeploy from code, rebuild indexes) for lower tiers; warm standby for critical agents. |
| **Model retirement** | Track retirement dates; evaluate replacements early (Topic 15 file 14). |
| **Testing** | Simulate a region or deployment outage in the gateway and measure user impact. |

## Hands-on

1. Put two model deployments in different regions behind an APIM backend pool with circuit breakers; load test until throttling triggers failover.
2. Enable Cosmos DB continuous backup and resource locks for a Standard-mode agent project.
3. Store agent definitions as code and redeploy them into a second project with a script.
4. Rebuild an AI Search index from source in a secondary region; measure time.
5. Write and drill a recovery runbook for "primary region model endpoint unavailable".

## Security, cost and reliability notes

- **Data residency**: Global deployments may process data anywhere the model is deployed; confirm policy before using them for failover.
- **Cost**: standby PTUs are expensive; prefer standard or Data Zone fallback capacity.
- **Security parity**: guardrails, private endpoints, identities and keys in every region.
- **Monitoring**: alert on 429s, circuit breaker trips and dependency health (Topic 26 file 09).

## Common mistakes

| Mistake | Fix |
|---|---|
| Single regional deployment with no fallback | Gateway backend pool across deployments/regions |
| Assuming Foundry fails over automatically | Design your own DR |
| No Cosmos DB backup for agent state | Continuous backup configured in advance |
| Agents built only in the portal | Definitions as code |
| Retries ignoring `Retry-After` | Respect it; use circuit breakers |

## Check yourself

1. Does Foundry provide automatic failover?
2. Who owns durability of Agent Service state in Standard mode?
3. How do APIM backend pools and circuit breakers improve model resilience?
4. What's on the Agent Service DR readiness checklist?
5. How do Global and Data Zone deployments help resilience?
6. How should you handle a vector index in DR?

<details>
<summary>Answers</summary>

1. No; you must design failover and DR.
2. The customer owns Cosmos DB, AI Search and Storage durability; Microsoft runs the control plane and runtime.
3. They distribute and prioritise traffic across deployments and stop sending to throttled or failing backends, honouring Retry-After.
4. Recovery strategy per project, baseline protections, agent definitions as code, automated redeploy, drills, monitoring alerts.
5. They route across Microsoft capacity in a broader scope, reducing dependence on one region's capacity (subject to residency).
6. Keep sources and pipelines in code to rebuild it, and for critical workloads maintain a second search service in another region.

</details>

## References

- [High availability and resiliency for Foundry projects and Agent Service](https://learn.microsoft.com/azure/foundry/how-to/high-availability-resiliency)
- [Foundry Agent Service disaster recovery](https://learn.microsoft.com/azure/foundry/how-to/agent-service-disaster-recovery)
- [Agent Service platform outage recovery](https://learn.microsoft.com/azure/foundry/how-to/agent-service-platform-disaster-recovery)
- [API Management backends and pools](https://learn.microsoft.com/azure/api-management/backends)
- [Azure OpenAI gateway with multiple backends](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
