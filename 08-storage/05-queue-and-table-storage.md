# Queue and Table Storage

## Why it matters

Queue Storage and Table Storage are the simplest, cheapest messaging and NoSQL options in Azure, bundled into every general-purpose storage account. They are good enough for a surprising number of workloads, from background job queues to device state lookups. Knowing their limits tells you when to stop and reach for Service Bus or Cosmos DB instead, which is a decision architects make often.

## Key concepts

### Queue Storage

A queue holds messages for asynchronous processing between components.

| Property | Detail |
|---|---|
| Message size | Up to 64 KB (store larger payloads in Blob and enqueue a reference) |
| Queue size | Up to the account capacity: millions of messages |
| Time to live | Default 7 days; configurable, including never expire (`-1`) |
| Delivery | **At least once**. A dequeued message becomes invisible for a visibility timeout; if not deleted in time, it reappears |
| Ordering | Best effort, no FIFO guarantee |
| Poison messages | `dequeueCount` property; you implement moving to a poison queue (Functions does it automatically) |
| Throughput | Thousands of messages per second per queue |
| Access | REST, SDKs, Entra RBAC (`Storage Queue Data Contributor`, `Message Sender`, `Message Processor`) |

The processing loop:

```text
receive (message becomes invisible for N seconds)
  → process
  → delete (with pop receipt)       ← success
  ✗ crash or timeout → message reappears, dequeueCount+1 → retry or poison
```

Because delivery is at least once, processing must be **idempotent**.

### Table Storage

A key-value and wide-column store for structured, schemaless entities.

| Property | Detail |
|---|---|
| Entity | Set of properties, up to 1 MB, up to 255 properties |
| Keys | **PartitionKey** + **RowKey** form the unique primary key |
| Indexing | Only the primary key. Queries on other properties scan |
| Transactions | Entity group transactions within one partition (up to 100 entities) |
| Scale | Partitions are spread across servers; a single partition has throughput limits |
| Query | OData filters; no joins, no secondary indexes, no aggregations |
| Pricing | Very cheap capacity and transactions |

Design tables around queries: choose PartitionKey and RowKey so your most common queries are point lookups (`PK + RK`) or partition range scans.

## How it works on Azure

### Queue with Python (keyless)

```python
import json
from azure.identity import DefaultAzureCredential
from azure.storage.queue import QueueClient

q = QueueClient("https://<account>.queue.core.windows.net", "ingest", credential=DefaultAzureCredential())
q.create_queue()
q.send_message(json.dumps({"blob": "docs/hr/handbook.pdf"}))

for msg in q.receive_messages(messages_per_page=16, visibility_timeout=120):
    job = json.loads(msg.content)
    if msg.dequeue_count > 5:
        # move to a poison queue and stop retrying
        QueueClient(q.url.rsplit("/", 1)[0], "ingest-poison", credential=DefaultAzureCredential()).send_message(msg.content)
        q.delete_message(msg)
        continue
    print("processing", job["blob"])   # must be idempotent
    q.delete_message(msg)
```

Azure Functions and Container Apps jobs (KEDA) can trigger on queues directly, handling polling, scaling and poison queues for you.

### Table with Python

```python
from azure.identity import DefaultAzureCredential
from azure.data.tables import TableServiceClient

svc = TableServiceClient("https://<account>.table.core.windows.net", credential=DefaultAzureCredential())
t = svc.create_table_if_not_exists("devicestate")
t.upsert_entity({"PartitionKey": "site-ams", "RowKey": "sensor-0042", "temp": 21.4, "status": "ok"})
print(t.get_entity("site-ams", "sensor-0042"))
for e in t.query_entities("PartitionKey eq 'site-ams' and status eq 'alarm'"):
    print(e["RowKey"])
```

The same code works against **Azure Cosmos DB for Table** by changing the endpoint, which adds global distribution, automatic secondary indexing, lower latency guarantees and higher throughput at higher cost.

## Design decisions and trade-offs

### Queue Storage vs Service Bus

| Need | Queue Storage | Service Bus |
|---|---|---|
| Simple background jobs | ✓ | ✓ |
| Very low cost, huge backlog | ✓ | – |
| Messages > 64 KB | – (use claim-check) | ✓ (256 KB standard, up to 100 MB premium) |
| FIFO ordering, sessions | – | ✓ |
| Duplicate detection | – | ✓ |
| Dead-lettering built in | Manual | ✓ |
| Transactions, scheduled delivery, deferral | – | ✓ |
| Topics and subscriptions (pub/sub) | – | ✓ |
| AMQP, JMS | – | ✓ |
| Server-side logs of message history | ✓ (storage logs) | Partial |

Use Queue Storage for simple, high-volume, order-insensitive work queues. Use Service Bus for enterprise messaging (topic 11).

### Table Storage vs Cosmos DB

| Need | Table Storage | Cosmos DB (for Table or NoSQL) |
|---|---|---|
| Cheapest key-value store | ✓ | – |
| Queries on non-key properties | Scans | Automatic indexing |
| Guaranteed single-digit ms latency, SLA on throughput | – | ✓ |
| Global distribution, multi-region writes | – (GRS only) | ✓ |
| Vector search, change feed, rich query | – | ✓ (NoSQL API) |

Use Table Storage for simple lookups, configuration, logs of state, and low-budget metadata. Use Cosmos DB when latency, query flexibility or global scale matters (topic 09).

## Hands-on

1. Create a storage account and assign yourself `Storage Queue Data Contributor` and `Storage Table Data Contributor`.
2. Run the queue sample. Then change the processing to raise an exception and observe `dequeue_count` increasing as the message reappears after the visibility timeout.
3. Run the table sample. Insert 1,000 entities across five partitions and compare the time of a point lookup with a query on `status` alone.
4. Create a Function with a queue trigger (topic 10) and watch it move a failing message to `<queue>-poison` after five attempts.

```bash
az storage message put --queue-name ingest --content "hello" --account-name $SA --auth-mode login
az storage message peek --queue-name ingest --account-name $SA --auth-mode login
az storage entity query --table-name devicestate --filter "PartitionKey eq 'site-ams'" --account-name $SA --auth-mode login
```

## Security, cost and reliability notes

- **Security:** use Entra data roles; disable shared key access; private endpoints for `queue` and `table` sub-resources.
- **Security:** never put sensitive payloads directly in messages that many components can read; reference secure storage instead.
- **Cost:** both services cost very little; transaction volume is the main driver. Long polling loops with empty receives cost transactions; use back-off or event-driven triggers.
- **Reliability:** design idempotent consumers; at-least-once means duplicates will happen.
- **Reliability:** Table Storage hot partitions throttle. Spread load across partition keys.

## Common mistakes

- Assuming queue messages arrive in order or exactly once.
- Setting a visibility timeout shorter than processing time, causing duplicate processing.
- No poison message handling, so a bad message loops forever.
- Putting large payloads in messages instead of using the claim-check pattern.
- Designing tables like relational tables and querying non-key properties at scale.
- A single partition key for all entities.

## Check yourself

1. What is the maximum message size in Queue Storage and how do you handle larger payloads?
2. What happens if a consumer crashes after receiving a message?
3. Why must queue consumers be idempotent?
4. What forms the primary key in Table Storage, and why does it matter for query design?
5. Give three reasons to choose Service Bus over Queue Storage.
6. When would you move from Table Storage to Cosmos DB?

<details>
<summary>Answers</summary>

1. 64 KB. Store the payload in Blob Storage and put a reference in the message (claim-check pattern).
2. The message becomes visible again after the visibility timeout, and another consumer can receive it; the dequeue count increases.
3. Delivery is at least once, so the same message can be processed more than once.
4. PartitionKey plus RowKey; they are the only indexed properties, so efficient queries must use them.
5. Any three of: FIFO and sessions, duplicate detection, built-in dead-lettering, larger messages, transactions, scheduled delivery, topics and subscriptions, AMQP and JMS.
6. When you need low-latency guarantees, secondary indexes, global distribution, multi-region writes, richer queries or vector search.

</details>

## References

- [Queue Storage introduction](https://learn.microsoft.com/azure/storage/queues/storage-queues-introduction)
- [Storage queues and Service Bus queues compared](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted)
- [Table Storage overview](https://learn.microsoft.com/azure/storage/tables/table-storage-overview)
- [Table design guide](https://learn.microsoft.com/azure/storage/tables/table-storage-design-guide)
- [Azure Cosmos DB for Table](https://learn.microsoft.com/azure/cosmos-db/table/introduction)
- [Claim-check pattern](https://learn.microsoft.com/azure/architecture/patterns/claim-check)
