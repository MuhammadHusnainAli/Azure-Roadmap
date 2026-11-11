# Mission-Critical Design

## Why it matters

Some workloads can't fail: payment processing, emergency services, trading platforms, airline operations, national digital services. For them, minutes of downtime mean large financial losses, safety risks or regulatory breaches. Microsoft's **mission-critical** guidance in the Well-Architected Framework, with reference implementations on Azure, describes how to approach very high availability (99.99% and beyond): **deployment stamps** in active-active regions, **health modelling** that understands the whole application, **zero-downtime deployments** with blue-green stamps, chaos and failure testing, and operations designed for rapid recovery. Even if your workload isn't mission-critical, these principles improve any important system.

## Key concepts

### Design principles

| Principle | Meaning |
|---|---|
| **Maximum reliability** | Design for failure at every layer; active-active across regions where justified |
| **Sustainable performance and scalability** | Scale units that scale out horizontally; validated capacity |
| **Operations by design** | Automation, observability and health modelling built in |
| **Cloud-native design** | PaaS, managed services, IaC |
| **Always secure** | Security integrated without compromising availability |

### Design areas

| Area | Key ideas |
|---|---|
| **Application platform** | Global resources (Front Door, Cosmos DB global, container registry) + regional **stamps** (compute, caches, queues) |
| **Application design** | Stateless compute, async messaging, idempotency, graceful degradation, retries/circuit breakers |
| **Data platform** | Globally distributed, multi-region write capable stores (Cosmos DB) or well-tested failover; consistency choices |
| **Networking and connectivity** | Global load balancing, private connectivity, DDoS, health probes |
| **Health modelling and observability** | Layered health model mapping component signals to user-flow health |
| **Deployment and testing** | Blue-green at stamp level, automated validation, chaos and load testing in pipelines |
| **Security** | Zero Trust, automated secret rotation, least privilege |
| **Operational procedures** | Runbooks, automated responses, incident management |

### Deployment stamps

Each stamp is a complete, independent copy of the regional application (scale unit). Stamps are **ephemeral**: new stamps are deployed for releases (blue-green), traffic shifts gradually via Front Door, and old stamps are removed. Benefits: limited blast radius, horizontal scale, consistent immutable deployments.

### Health modelling

A **health model** defines how component signals (metrics, logs, dependency checks) roll up into the health of user flows and the overall application, with thresholds for healthy, degraded and unhealthy. Front Door routes away from unhealthy stamps based on composite health endpoints; dashboards show health by flow and stamp.

### Zero-downtime deployment

```text
Build once → deploy new stamp(s) (green) alongside existing (blue) → automated tests and chaos/load validation
→ Front Door shifts traffic gradually (10% → 50% → 100%) with health gates → decommission blue stamps
Rollback = shift traffic back to blue stamps (kept until confidence)
```

Infrastructure and application are deployed together per stamp; no in-place changes in production.

## How it works on Azure

Reference topology (based on Microsoft's mission-critical reference implementation):

```text
Global resources: Azure Front Door Premium (WAF, health probes), Cosmos DB (multi-region writes), ACR (geo-replicated), Log Analytics (global)
Regional stamp (×N per region, ×2+ regions): AKS (or Container Apps), Event Hubs, Key Vault, Storage, Redis, regional Log Analytics/App Insights
Pipelines: Azure DevOps/GitHub Actions deploying full stamps with Terraform/Bicep; automated E2E, load and chaos tests per release
Health: Health service per stamp aggregating dependency checks → Front Door probe endpoint; health model dashboards in Azure Monitor
```

Composite health endpoint (sketch):

```python
@app.get("/health/stamp")
def stamp_health():
    checks = {"cosmos": check_cosmos(), "eventhubs": check_eventhubs(), "keyvault": check_keyvault(), "redis": check_redis()}
    healthy = all(c["ok"] for c in checks.values())
    return JSONResponse(status_code=200 if healthy else 503, content={"healthy": healthy, "checks": checks})
```

Keep health checks fast and cached to avoid overloading dependencies.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Is it really mission-critical?** | Validate with business impact; cost and complexity are high. |
| **Active-active vs active-passive** | Active-active for near-zero RTO; requires multi-write data or careful partitioning. |
| **Data consistency** | Cosmos DB consistency levels vs latency; design for eventual consistency where acceptable. |
| **Stamp size** | Balance blast radius, cost and operational overhead. |
| **Automation** | Fully automated deployments and recovery; manual steps are failure points. |
| **Dependencies** | Every dependency (identity, DNS, third parties) must meet or exceed target availability, or be isolated with fallbacks. |

## Hands-on

1. Review Microsoft's mission-critical reference implementation and diagram its global and regional resources.
2. Build a composite health endpoint and use it as a Front Door probe.
3. Implement blue-green stamp deployment with gradual Front Door weight shifting.
4. Draft a health model for three user flows.
5. Run a chaos experiment removing a region under load (Topic 28 file 07).

## Security, cost and reliability notes

- **Cost**: multiple active regions and stamps multiply infrastructure costs; justify with business impact.
- **Security** must not introduce single points of failure (for example, a single-region Key Vault dependency).
- **Operational load**: requires mature SRE practices and on-call.
- **Testing**: continuous validation in pre-production and production.

## Common mistakes

| Mistake | Fix |
|---|---|
| Labelling workloads mission-critical without business justification | Criticality assessment |
| In-place production changes | Immutable stamps |
| Health probes checking only "process is up" | Composite, dependency-aware health |
| Single-region dependencies in an active-active design | Map and remove |
| Manual failover procedures | Automated, tested failover |

## Check yourself

1. List the mission-critical design principles.
2. What is a deployment stamp, and why make stamps ephemeral?
3. What is health modelling?
4. Describe zero-downtime deployment with stamps.
5. Which resources are global vs regional in the reference architecture?
6. What questions should you ask before adopting mission-critical design?

<details>
<summary>Answers</summary>

1. Maximum reliability, sustainable performance and scalability, operations by design, cloud-native design, always secure.
2. An independent, complete regional copy of the application; ephemeral stamps enable blue-green releases and limit drift and blast radius.
3. Defining how component signals roll up into user-flow and application health with thresholds.
4. Deploy new stamps alongside old, validate automatically, shift traffic gradually via Front Door with health gates, remove old stamps; roll back by shifting back.
5. Global: Front Door, Cosmos DB, ACR, global logs. Regional: compute (AKS/Container Apps), messaging, Key Vault, storage, cache, regional monitoring.
6. Business impact of downtime, budget, team maturity, dependency availability, data consistency needs.

</details>

## References

- [Mission-critical workloads overview](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-overview)
- [Health modelling](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-health-modeling)
- [Deployment and testing for mission-critical](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-deployment-testing)
- [Mission-critical baseline architecture](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-mission-critical/mission-critical-intro)
- [Deployment stamps pattern](https://learn.microsoft.com/azure/architecture/patterns/deployment-stamp)
