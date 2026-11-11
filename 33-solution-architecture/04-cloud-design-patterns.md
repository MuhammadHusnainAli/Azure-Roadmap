# Cloud Design Patterns

## Why it matters

Distributed systems fail in recurring ways: transient network errors, overloaded dependencies, cascading failures, contention between reads and writes, slow remote calls, chatty clients. **Cloud design patterns** are proven, named solutions to these problems. Knowing them lets architects design resilient, scalable systems quickly and communicate designs precisely ("put a circuit breaker on the payment call and use queue-based load levelling for order processing"). The Azure Architecture Center catalogues them with context, trade-offs and Azure implementation examples, and they appear throughout WAF guidance and the AZ-305 exam.

## Key concepts

### Resilience patterns

| Pattern | Problem | Solution | Azure examples |
|---|---|---|---|
| **Retry** | Transient faults | Retry with exponential backoff and jitter, bounded attempts | Azure SDK retry policies, Polly, Durable Functions retries |
| **Circuit breaker** | Repeated calls to a failing dependency waste resources and cascade failures | Stop calling after failures; probe periodically; fail fast | Polly, APIM backend circuit breakers, Dapr resiliency |
| **Bulkhead** | One component's failure exhausts shared resources | Isolate resources (pools, instances, connection limits) per consumer/dependency | Separate App Service plans, AKS node pools, connection pool limits |
| **Health endpoint monitoring** | Need to know if an app is healthy | Expose health endpoints checked by load balancers and monitors | App Service health check, Front Door probes, Kubernetes probes |
| **Queue-based load levelling** | Spiky load overwhelms services | Buffer requests in a queue; process at a sustainable rate | Service Bus, Storage queues, KEDA scaling |
| **Throttling / rate limiting** | Protect services from overload or abuse | Limit per client/tenant | APIM rate limits, token limits for AI |
| **Compensating transaction** | Undo multistep operations without distributed transactions | Compensating actions | Durable Functions, Logic Apps sagas |
| **Saga** | Data consistency across services | Sequence of local transactions with compensations (choreography or orchestration) | Service Bus, Durable Functions |
| **Leader election** | Coordinate a single active instance | Elect via leases | Blob leases, Kubernetes leases |

### Data and performance patterns

| Pattern | Problem | Solution | Azure examples |
|---|---|---|---|
| **Cache-aside** | Repeated reads of slow data | Load into cache on miss; invalidate on change | Azure Managed Redis |
| **CQRS** | Read and write models have different needs | Separate commands and queries, often with separate stores | Cosmos DB change feed to read models, Azure SQL + Search |
| **Event sourcing** | Need full history and audit | Store events as source of truth; rebuild state | Event Hubs, Cosmos DB |
| **Materialised view** | Complex queries are slow | Precompute views | Cosmos DB change feed, Fabric |
| **Sharding** | One store can't scale | Partition data | Cosmos DB partitions, Elastic database tools |
| **Static content hosting** | Serving static files from compute | Storage/CDN | Static Web Apps, Blob + Front Door |
| **Valet key** | Clients upload/download large data through your app | Short-lived scoped tokens | SAS (user delegation), signed URLs |
| **Claim check** | Large messages in queues | Store payload, pass reference | Blob + Service Bus |

### Gateway and integration patterns

| Pattern | Use | Azure |
|---|---|---|
| **Gateway routing** | Route requests to services via one endpoint | Front Door, Application Gateway, APIM |
| **Gateway aggregation** | Combine multiple backend calls | APIM policies, BFF services |
| **Gateway offloading** | Offload TLS, auth, rate limiting | APIM, Front Door, App Gateway |
| **Backends for Frontends (BFF)** | Tailored APIs per client type | Separate APIs/Functions per frontend |
| **Strangler fig** | Incremental migration | APIM/Front Door routing (Topic 30) |
| **Anti-corruption layer** | Isolate legacy semantics | Adapter services |
| **Ambassador / sidecar** | Cross-cutting concerns next to the app | Dapr, service mesh sidecars |
| **Publisher-subscriber** | Decouple producers and consumers | Event Grid, Service Bus topics |
| **Competing consumers** | Scale message processing | Service Bus queues with multiple workers |

### Deployment patterns

**Deployment stamps** (scale units), **geode** (globally distributed nodes serving any request), and **federated identity**. AI-specific patterns (gateway for models, RAG, agent orchestration) appear in Topic 34.

## How it works on Azure

Retry and circuit breaker with the Azure SDK and Polly (.NET):

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new() { MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true })
    .AddCircuitBreaker(new() { FailureRatio = 0.5, MinimumThroughput = 20,
                               SamplingDuration = TimeSpan.FromSeconds(30), BreakDuration = TimeSpan.FromSeconds(15) })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();

var response = await pipeline.ExecuteAsync(async ct => await http.GetAsync("https://payments/api/charge", ct));
```

Cache-aside with Redis (Python):

```python
import json, redis
r = redis.Redis(host="redis-contoso.westeurope.redis.azure.net", port=10000, ssl=True, password=token)  # Entra token-based auth in production

def get_product(pid):
    cached = r.get(f"product:{pid}")
    if cached:
        return json.loads(cached)
    product = db_load_product(pid)
    r.setex(f"product:{pid}", 300, json.dumps(product))
    return product
```

Queue-based load levelling: front-end writes orders to a Service Bus queue; Container Apps workers scale with KEDA on queue length; idempotent processing with duplicate detection.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Retries** | Only for transient faults; idempotent operations; cap total time; combine with circuit breakers. |
| **Caching** | Define staleness tolerance; plan invalidation; protect against stampedes. |
| **CQRS/event sourcing** | Powerful but complex; use only where read/write asymmetry or audit needs justify it. |
| **Gateways** | Centralise cross-cutting concerns; avoid putting business logic in gateways. |
| **Async messaging** | Improves resilience and scale; adds eventual consistency and operational complexity. |
| **Pattern stacking** | Combine patterns deliberately; document why. |

## Hands-on

1. Add retry, timeout and circuit breaker to a service call; simulate failures and observe behaviour.
2. Implement cache-aside with Azure Managed Redis and measure latency.
3. Build queue-based load levelling with Service Bus and KEDA-scaled workers.
4. Implement the valet key pattern for direct uploads with user delegation SAS.
5. Use APIM for gateway offloading of authentication and rate limiting.

## Security, cost and reliability notes

- **Retry storms** can amplify outages; use jitter, limits and circuit breakers.
- **Caches** may hold sensitive data; secure and expire it.
- **Valet keys** must be short-lived and narrowly scoped.
- **Cost**: queues, caches and gateways add services; justify by reliability/performance gains.

## Common mistakes

| Mistake | Fix |
|---|---|
| Infinite or immediate retries | Exponential backoff with jitter and limits |
| Retrying non-idempotent operations | Idempotency keys |
| CQRS everywhere | Use where justified |
| Business logic in API gateways | Keep in services |
| No timeouts | Explicit timeouts on every remote call |

## Check yourself

1. Explain retry, circuit breaker and bulkhead and how they combine.
2. When would you use queue-based load levelling?
3. What problem does CQRS solve, and what does it cost?
4. Describe cache-aside.
5. What are gateway routing, aggregation and offloading?
6. What is the valet key pattern?

<details>
<summary>Answers</summary>

1. Retry handles transient faults; circuit breaker stops calling failing dependencies; bulkhead isolates resources; together they prevent cascading failures.
2. When load is spiky and backends need protection or can process asynchronously.
3. Separates read and write models for different scaling/shape needs; costs complexity and eventual consistency.
4. Read from cache; on miss load from the store and populate the cache; invalidate or expire on change.
5. Routing: one endpoint to many services; aggregation: combine backend calls; offloading: move cross-cutting concerns (TLS, auth, rate limits) to the gateway.
6. Give clients short-lived, scoped tokens to access storage directly instead of proxying data through the app.

</details>

## References

- [Cloud design patterns](https://learn.microsoft.com/azure/architecture/patterns/)
- [Retry pattern](https://learn.microsoft.com/azure/architecture/patterns/retry)
- [Circuit breaker pattern](https://learn.microsoft.com/azure/architecture/patterns/circuit-breaker)
- [Bulkhead pattern](https://learn.microsoft.com/azure/architecture/patterns/bulkhead)
- [CQRS pattern](https://learn.microsoft.com/azure/architecture/patterns/cqrs)
- [Cache-aside pattern](https://learn.microsoft.com/azure/architecture/patterns/cache-aside)
- [Gateway routing pattern](https://learn.microsoft.com/azure/architecture/patterns/gateway-routing)
