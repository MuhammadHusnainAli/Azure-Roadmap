# Architecture Styles

## Why it matters

Before choosing services, an architect chooses a **style**: the overall shape of the system and the constraints it imposes. An N-tier web app, a set of microservices, an event-driven pipeline and a big data platform behave very differently in how they scale, fail, evolve and cost. Choosing the wrong style (for example microservices for a small team with a simple domain) creates years of unnecessary complexity; choosing too simple a style for a fast-growing platform creates a rewrite. The Azure Architecture Center describes the main styles, their benefits, challenges and best-fit Azure services, giving architects a structured starting point.

## Key concepts

### Main styles

| Style | Shape | Strengths | Challenges | Typical Azure services |
|---|---|---|---|---|
| **N-tier** | Logical layers (presentation, business, data) on separate tiers | Familiar, simple migration path, works for lift-and-shift | Monolithic deployment, scaling granularity, tight coupling | VMs/VMSS, App Service, Azure SQL, Application Gateway |
| **Web-queue-worker** | Web front end, message queue, background worker | Simple, separates long-running work, scales independently | Front end and worker can grow into monoliths | App Service + WebJobs/Functions, Container Apps, Service Bus/Storage queues, Redis |
| **Microservices** | Small, autonomous services owning their data, communicating via APIs/events | Independent deployment and scaling, team autonomy, technology flexibility | Distributed complexity: data consistency, observability, testing, ops maturity required | AKS, Container Apps, APIM, Service Bus, Cosmos DB, Dapr |
| **Event-driven** | Producers emit events; consumers react (pub/sub or event streaming) | Loose coupling, real-time processing, scale | Eventual consistency, ordering, debugging, schema evolution | Event Grid, Event Hubs, Service Bus, Functions, Stream Analytics, Fabric Real-Time Intelligence |
| **Big data / analytics** | Batch and stream processing over large datasets, lakehouse | Scale, analytics and ML | Complexity, governance, cost | Fabric, Databricks, ADLS, Synapse (legacy), Data Factory |
| **Big compute (HPC)** | Many parallel compute tasks | Massive parallelism | Scheduling, cost, data movement | Azure Batch, HPC VMs, CycleCloud |
| **Serverless** (cross-cutting) | Functions and managed services, pay per execution | No infrastructure, scale to zero | Cold starts, limits, vendor-specific | Functions, Logic Apps, Container Apps, Event Grid |

AI solutions combine these: RAG apps are often N-tier/web-queue-worker plus event-driven ingestion; agentic systems add orchestration and tool services (Topic 34).

### Selection criteria

| Question | Implication |
|---|---|
| Team size and maturity? | Small teams: modular monolith or web-queue-worker; microservices need DevOps maturity |
| Domain complexity and change rate? | Many bounded contexts changing independently favour microservices |
| Scale profile? | Spiky, asynchronous workloads favour queues/events |
| Consistency needs? | Strong consistency favours fewer services and shared transactions |
| Latency? | Synchronous chains add latency; prefer async where possible |
| Operational capability? | Observability, CI/CD and on-call readiness before distributed styles |

### Modular monolith

A single deployable with strong internal module boundaries is often the best starting point: simple to operate, easy to split later along well-defined seams (strangler fig).

## How it works on Azure

Reference shapes:

```text
N-tier:              Front Door → App Gateway WAF → web VMSS/App Service → API tier → Azure SQL (zone-redundant)
Web-queue-worker:    App Service (web) → Service Bus queue → Container Apps worker (KEDA) → Cosmos DB / Blob
Microservices:       Front Door → APIM → Container Apps/AKS services (each with own DB) ↔ Service Bus/Event Grid; Dapr; OpenTelemetry
Event-driven:        IoT/apps → Event Hubs → Stream Analytics/Fabric RTI → alerts, dashboards; Event Grid → Functions reactions
Big data:            Sources → Data Factory/Fabric pipelines → OneLake/ADLS (medallion) → Spark/Warehouse → Power BI / ML
```

Choosing between Container Apps and AKS for microservices: Container Apps for most teams (managed, Dapr, KEDA, revisions); AKS for advanced networking, custom operators or large platform teams ([Topic 07](../07-containers-and-kubernetes/README.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Start simple** | Modular monolith or web-queue-worker unless strong drivers for microservices. |
| **Data ownership** | Microservices own their data; avoid shared databases across services. |
| **Sync vs async** | Prefer async messaging between services for resilience; sync for queries needing immediate answers. |
| **Observability first** | Distributed tracing is mandatory for microservices and event-driven systems. |
| **Evolution** | Design seams and contracts so styles can evolve. |

## Hands-on

1. Classify three existing applications by style and list their pain points.
2. Design the same e-commerce system as N-tier, web-queue-worker and microservices; compare cost, complexity and scalability.
3. Implement a web-queue-worker sample with App Service, Service Bus and Container Apps.
4. Build an event-driven flow with Event Grid and Functions.
5. Write an ADR choosing a style for a new product.

## Security, cost and reliability notes

- **Distributed styles** increase attack surface (more endpoints, identities, secrets); use managed identity and gateways.
- **Cost**: microservices duplicate infrastructure overheads; serverless can be cheap at low scale and expensive at high constant load.
- **Reliability**: async styles absorb failures better but need dead-letter handling and idempotency.

## Common mistakes

| Mistake | Fix |
|---|---|
| Microservices for a small team/simple domain | Modular monolith |
| Distributed monolith (services sharing a DB, chatty sync calls) | Data ownership and async messaging |
| Event-driven without schema management | Schema registry, versioning |
| No tracing in distributed systems | OpenTelemetry from day one |
| Picking a style for fashion | Criteria-based decision |

## Check yourself

1. Describe N-tier, web-queue-worker, microservices, event-driven and big data styles.
2. What are the main challenges of microservices?
3. When is web-queue-worker a good fit?
4. Which criteria guide style selection?
5. What is a modular monolith, and why start there?
6. Which Azure services typically support event-driven architectures?

<details>
<summary>Answers</summary>

1. N-tier: layered tiers; web-queue-worker: web front end plus queue and background worker; microservices: autonomous services owning data; event-driven: producers and consumers via events; big data: batch/stream processing over large datasets.
2. Distributed data consistency, observability, testing, deployment and operational maturity.
3. Apps with background or long-running work and a relatively simple domain.
4. Team size/maturity, domain complexity and change rate, scale profile, consistency, latency, operational capability.
5. A single deployable with strong internal module boundaries; simple to run and easy to split later.
6. Event Grid, Event Hubs, Service Bus, Functions, Stream Analytics, Fabric Real-Time Intelligence.

</details>

## References

- [Architecture styles](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/)
- [N-tier style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/n-tier)
- [Web-queue-worker style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/web-queue-worker)
- [Microservices style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/microservices)
- [Event-driven style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/event-driven)
- [Big data style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/big-data)
