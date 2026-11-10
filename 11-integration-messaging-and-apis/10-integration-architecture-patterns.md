# Integration Architecture Patterns

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

The services in this topic are building blocks. Patterns are how you combine them into systems that keep working when networks fail, messages are duplicated, and services are slow. A few patterns come up again and again in Azure designs and architecture interviews: **event-driven architecture**, **competing consumers**, **queue-based load levelling**, **claim check**, **transactional outbox**, **saga** and **idempotent receiver**. They matter even more for AI systems, where calls are slow, rate-limited, expensive and often long-running.

## Key concepts

### Event-driven architecture

Producers publish events about state changes; consumers react independently. Producers don't know who consumes.

| Style | Description | Azure services |
|---|---|---|
| **Pub-sub (event notification)** | Event says "X happened"; subscribers fetch details if needed | Event Grid, Service Bus topics |
| **Event-carried state transfer** | Event carries the data consumers need | Service Bus, Event Hubs |
| **Event streaming** | Ordered, replayable log consumed by many readers | Event Hubs (Kafka) |
| **Event sourcing** | The log of events *is* the source of truth; state is derived | Cosmos DB change feed, Event Hubs |

Benefits: loose coupling, independent scaling, easy to add consumers. Costs: eventual consistency, harder debugging (use distributed tracing and correlation IDs), and the need for idempotency.

### Pattern catalogue

| Pattern | Problem | Solution | Azure implementation |
|---|---|---|---|
| **Competing consumers** | One consumer can't keep up | Many consumers read from the same queue; each message goes to one | Service Bus queue + Functions / Container Apps with KEDA scaling |
| **Queue-based load levelling** | Spiky load overwhelms a rate-limited back end | A queue buffers requests; consumers process at a sustainable rate | Service Bus or Storage queue in front of model or embedding calls |
| **Claim check** | Payloads too large or sensitive for the broker | Store the payload in storage, send only a reference | Blob storage + Service Bus or Event Grid message with blob URL |
| **Transactional outbox** | Must update a database *and* publish a message atomically | Write the message to an outbox table in the same DB transaction; a relay publishes it | SQL/Cosmos DB outbox + change feed or a relay function → Service Bus |
| **Saga** | A business transaction spans services with no distributed transaction | A sequence of local transactions with **compensating actions** on failure | Durable Functions orchestration, Logic Apps, or choreography via Service Bus topics |
| **Idempotent receiver / inbox** | Messages are delivered more than once | Record processed message IDs; ignore repeats | Service Bus duplicate detection + an inbox table |
| **Publisher-subscriber** | Many consumers need the same message | Topic with subscriptions | Service Bus topics, Event Grid |
| **Priority queue** | Some requests are more urgent | Separate queues or subscriptions by priority, with more consumers on high | Service Bus topic with filtered subscriptions |
| **Sequential convoy** | Order matters per group but groups can run in parallel | Partition by a key; process each group in order | Service Bus **sessions**, Event Hubs partition key |
| **Asynchronous request-reply** | Long-running work behind an HTTP API | Return 202 with a status URL; client polls or gets a callback | Durable Functions HTTP API, Logic Apps, APIM |
| **Circuit breaker** | A failing dependency causes cascading failures | Stop calling it for a while after failures | APIM backend circuit breaker, Polly / resilience libraries |
| **Retry with backoff** | Transient failures | Retry with exponential backoff and jitter | Azure SDK retry policies, APIM `retry` |
| **Strangler fig** | Replace a legacy system gradually | Route calls through a façade; move features one by one | APIM or Front Door as the façade (see topic 10) |

## How it works on Azure

### Transactional outbox with Cosmos DB

The **dual-write problem**: saving an order and publishing `OrderCreated` are two systems. If the second step fails, they disagree.

```text
API ──► Cosmos DB transactional batch (same partition key):
          1. upsert  order document
          2. create  outbox document { type: "OrderCreated", payload, status: "pending" }
Change feed processor (Function with Cosmos DB trigger)
          └─► Service Bus topic "orders"  (MessageId = outbox id → duplicate detection)
          └─► mark outbox document as sent (or let TTL delete it)
```

```python
# Inside the API, write the order and its outbox record atomically (same partition key)
batch = [
    ("upsert", (order,)),
    ("create", ({"id": f"outbox-{order['id']}", "pk": order["pk"], "type": "OrderCreated",
                 "payload": order, "ttl": 86400},)),
]
container.execute_item_batch(batch_operations=batch, partition_key=order["pk"])
```

With SQL, use an `Outbox` table in the same database transaction and a relay that polls it or uses change tracking or CDC.

### Saga: orchestration vs choreography

| | Orchestration | Choreography |
|---|---|---|
| Control | A central orchestrator tells each service what to do and runs compensations | Each service reacts to events and emits its own |
| Visibility | One place to see state | State spread across services |
| Coupling | Orchestrator knows all participants | Services know only events |
| Azure | **Durable Functions** (or Durable Task Scheduler), Logic Apps | Service Bus topics, Event Grid |
| Best for | Complex flows, clear compensation logic, human steps | Simple flows with few steps |

```python
# Durable Functions orchestrator: travel booking saga with compensation
import azure.durable_functions as df

def orchestrator(ctx: df.DurableOrchestrationContext):
    booking = ctx.get_input()
    done = []
    try:
        yield ctx.call_activity("ReserveFlight", booking); done.append("CancelFlight")
        yield ctx.call_activity("ReserveHotel", booking);  done.append("CancelHotel")
        yield ctx.call_activity("ChargeCard", booking);    done.append("RefundCard")
        return "confirmed"
    except Exception:
        for compensation in reversed(done):
            yield ctx.call_activity(compensation, booking)
        return "compensated"

main = df.Orchestrator.create(orchestrator)
```

Compensations must themselves be **idempotent** and retryable. A saga provides eventual consistency, not isolation, so other transactions may see intermediate states.

### Claim check for documents and AI payloads

```text
Producer: upload document → Blob (container "payloads", private)
          send Service Bus message { "blobUrl": ".../payloads/abc.pdf", "sha256": "...", "contentType": "application/pdf" }
Consumer: read message → fetch blob with managed identity → process → delete or tier the blob
```

Event Grid's `BlobCreated` events are already claim checks: they carry the blob URL, not the content.

### Competing consumers with KEDA

```text
Service Bus queue "embed-jobs"  ──► Container Apps job / app
  scale rule: azure-servicebus, messageCount=50 per replica, min 0, max 20 replicas
  each replica: receive batch → call embeddings (≤ quota) → index → complete
```

Cap the maximum replicas so total throughput stays within your model TPM quota. That makes it queue-based load levelling and competing consumers together.

### Patterns for AI and agent systems

| Situation | Pattern |
|---|---|
| User asks an agent to do a 10-minute task | Asynchronous request-reply: API enqueues, returns 202 + status URL; worker processes; push completion via Web PubSub or SignalR |
| Bursty requests against a fixed TPM quota | Queue-based load levelling + competing consumers + retry with backoff |
| Agent tool that books, pays or emails | Saga with compensation, idempotency keys, human approval step |
| Ingesting large documents | Claim check + event-driven ingestion |
| Agent must notify other systems after writing state | Transactional outbox |
| Multi-step agent workflow that must survive restarts | Durable orchestration (Durable Functions, Durable Task Scheduler, or Foundry workflow agents) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Orchestration vs choreography | Orchestrate when there are more than three or four steps, compensations or human tasks; choreograph simple reactive flows |
| Exactly-once vs at-least-once | Assume at-least-once everywhere; build idempotent consumers rather than chasing exactly-once delivery |
| Outbox vs "publish then save" | Use an outbox whenever losing or falsely emitting an event has business impact |
| Claim check threshold | Use it when payloads approach broker limits (256 KB on Service Bus Standard) or contain sensitive data you don't want copied into broker logs |
| Event payload size | Thin events (IDs) reduce coupling and size but cause call-backs; fat events reduce call-backs but leak schema. Version events explicitly |
| Ordering | Require it only where the business needs it, and scope it by key (session or partition), never globally |

## Hands-on

1. Build the **outbox** with Cosmos DB: an HTTP Function writes an order and outbox record in a transactional batch; a Cosmos DB-triggered function publishes to a Service Bus topic with `MessageId` set to the outbox ID and duplicate detection on.
2. Build the **saga** above with Durable Functions. Make `ChargeCard` fail and confirm compensations run in reverse.
3. Implement **claim check**: upload a 5 MB file to Blob, send a reference via Service Bus Standard (256 KB limit), and process it in a consumer.
4. Implement **competing consumers** with a Container Apps job scaled by a KEDA Service Bus rule. Send 1,000 messages and watch replicas scale out and back to zero.
5. Draw the architecture for "user uploads a contract; an agent summarises it, flags risks, and requests legal approval", naming every pattern you used.

## Security, cost and reliability notes

- **Security:** claim check blobs must be private and accessed with managed identity or short-lived user delegation SAS; never put secrets in messages.
- **Security:** propagate the user's identity or correlation through events deliberately, and avoid copying personal data into many queues.
- **Cost:** fan-out multiplies operations, since every subscription and retry is billed on per-operation tiers. Event-driven designs can be cheap at rest (scale to zero) but costly when chatty.
- **Reliability:** idempotency everywhere. Use dead-letter queues and alert on them, and use correlation IDs with distributed tracing (OpenTelemetry, Application Insights) across producers and consumers.
- **Reliability:** test failure paths: kill consumers mid-message, inject duplicates, make downstream services return 429 and 500.

## Common mistakes

- Saving to the database and then publishing in two steps, with no outbox.
- Distributed transactions across services instead of sagas.
- Compensations that aren't idempotent, so a retry refunds twice.
- Global ordering requirements that serialise the whole system.
- Fat payloads in messages instead of claim checks.
- Auto-scaling consumers beyond what the downstream (for example the model quota) can handle.
- No correlation IDs, so incidents can't be traced across services.

## Check yourself

1. What problem does the transactional outbox solve?
2. How does a saga differ from a distributed transaction?
3. When would you choose choreography over orchestration?
4. Describe the claim check pattern and when to use it.
5. How do competing consumers and queue-based load levelling work together for an embeddings pipeline?
6. Why must message consumers be idempotent?
7. Which Azure features help implement the sequential convoy pattern?

<details>
<summary>Answers</summary>

1. The dual-write problem: updating a database and publishing a message atomically. Both are written in one local transaction and a relay publishes the message reliably.
2. A saga is a sequence of local transactions with compensating actions on failure, giving eventual consistency without locks across services. A distributed transaction coordinates a single atomic commit across resources.
3. For simple flows with few steps where services can react to each other's events without central coordination.
4. Store the large or sensitive payload in storage and send only a reference (plus a hash or metadata) in the message; the consumer fetches it. Use it when payloads approach broker limits or shouldn't flow through the broker.
5. A queue absorbs bursts (load levelling); multiple consumers process in parallel (competing consumers), with the maximum consumer count capped to stay within the embeddings quota.
6. Because messaging services deliver at least once; retries, lock expiry and failovers can deliver the same message more than once.
7. Service Bus sessions (FIFO per session ID) and Event Hubs partition keys (order per partition).

</details>

## References

- [Cloud design patterns (Architecture Center)](https://learn.microsoft.com/azure/architecture/patterns/)
- [Event-driven architecture style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/event-driven)
- [Competing consumers pattern](https://learn.microsoft.com/azure/architecture/patterns/competing-consumers)
- [Queue-based load leveling pattern](https://learn.microsoft.com/azure/architecture/patterns/queue-based-load-leveling)
- [Claim-check pattern](https://learn.microsoft.com/azure/architecture/patterns/claim-check)
- [Saga design pattern](https://learn.microsoft.com/azure/architecture/patterns/saga)
- [Transactional outbox with Azure Cosmos DB](https://learn.microsoft.com/azure/architecture/databases/guide/transactional-outbox-cosmos)
- [Sequential convoy pattern](https://learn.microsoft.com/azure/architecture/patterns/sequential-convoy)
- [Asynchronous request-reply pattern](https://learn.microsoft.com/azure/architecture/patterns/async-request-reply)
