# Azure Service Bus

## Why it matters

Service Bus is Azure's enterprise message broker. Use it when a message represents business intent (an order, a payment, a request for an agent to do work) and losing it, processing it twice or processing it out of order would hurt. It decouples producers from consumers, absorbs spikes, and gives you dead-lettering, sessions, transactions and duplicate detection without running your own broker. In AI systems, Service Bus often sits between a front end and long-running agent or ingestion workers.

## Key concepts

### Entities

| Entity | Purpose |
|---|---|
| **Namespace** | Container and DNS name (`<name>.servicebus.windows.net`); the unit of tier, networking and capacity |
| **Queue** | Point-to-point: each message is processed by one competing consumer |
| **Topic** | Publish-subscribe: a message sent to the topic is copied to every matching subscription |
| **Subscription** | A virtual queue on a topic, with **rules** (filters and actions) that decide which messages it receives |

### Receive modes

| Mode | Behaviour | Use when |
|---|---|---|
| **Peek-lock** (default) | Message is locked for the consumer; the consumer must **complete**, **abandon**, **defer** or **dead-letter** it. If the lock expires, the message becomes available again | You need at-least-once, safe processing |
| **Receive and delete** | Message is removed as soon as it is delivered | You can tolerate loss (at-most-once) and want speed |

The lock duration is set on the entity (default 1 minute, maximum 5 minutes). Long processing must **renew the lock** (the SDKs offer auto lock renewal).

### Dead-letter queue (DLQ)

Every queue and subscription has a sub-queue at `<entity>/$deadletterqueue`. Messages land there when:

| Reason | Trigger |
|---|---|
| `MaxDeliveryCountExceeded` | Delivered more than **MaxDeliveryCount** times (default 10) without completion |
| `TTLExpiredException` | Time to live expired and **dead-lettering on expiration** is enabled |
| Filter evaluation errors | A subscription rule failed to evaluate (if enabled) |
| Application dead-lettering | Your code explicitly dead-letters a poison message with a reason |

Messages in the DLQ never expire automatically. You must monitor, inspect and resubmit or discard them.

### Sessions

Sessions give **FIFO ordering and exclusive processing per session ID**. A receiver accepts a session and receives only that session's messages, in order, until it releases it. Each session can store **session state** (up to the message size limit), which is handy for workflow progress. Typical uses: all messages for one order, one customer or one **agent conversation**.

### Other broker features

| Feature | What it does |
|---|---|
| **Duplicate detection** | Drops messages with a `MessageId` already seen within a configurable window (default 10 minutes, 20 seconds to 7 days; Standard and Premium) |
| **Scheduled messages** | Enqueue now, become visible at `ScheduledEnqueueTimeUtc` |
| **Deferral** | Set a message aside and retrieve it later by sequence number |
| **Auto-forwarding** | Chain a queue or subscription to another queue or topic |
| **Transactions** | Group operations (sends and completes) on one entity, or across entities via a "send-via" entity, atomically; up to 100 messages per transaction |
| **Message TTL** | Expire stale messages; optionally dead-letter them |
| **Auto-delete on idle** | Delete an entity after a period of inactivity |
| **Filters** | SQL filters (up to 2,000 per topic) or correlation filters (up to 100,000 per topic) on subscriptions |

## How it works on Azure

### Tiers (checked 2026-11)

| | Basic | Standard | Premium |
|---|---|---|---|
| Queues | Yes | Yes | Yes |
| Topics and subscriptions | No | Yes | Yes |
| Sessions, transactions, duplicate detection | No | Yes | Yes |
| Capacity model | Shared, pay per operation | Shared, pay per operation (base charge plus operations) | Dedicated **messaging units** (1, 2, 4, 8 or 16 per namespace) |
| Max message size | 256 KB | 256 KB | 1 MB by default per entity, configurable up to **100 MB** over AMQP (1 MB over HTTP; batches up to 1 MB) |
| Queue or topic size | 1–5 GB (80 GB if partitioned) | 1–5 GB (80 GB if partitioned) | 80 GB per entity; namespace 1 TB per MU |
| Private endpoints, VNet service endpoints | No | No | Yes |
| Customer-managed keys | No | No | Yes |
| Geo-replication (data and metadata) | No | No | Yes |
| Geo-disaster recovery (metadata only) | No | No | Yes |
| JMS | No | JMS 1.1 subset (queues) | JMS 1.1 and 2.0 |
| Availability zones | Spread across zones in supported regions | Yes | Yes |

**Partitioning.** In Basic and Standard, partitioning is set per entity at creation and uses 16 partitions. In Premium, partitioning is set at **namespace** creation (1, 2 or 4 partitions) and applies to every entity. The number of MUs must be a multiple of the number of partitions. It can't be changed afterwards.

**Sizing Premium.** Start with 1–2 MUs. Microsoft's guidance is to scale up when CPU is above 75% or memory above 60%, and to consider scaling down when CPU is below 25%. MUs are billed hourly at the highest count during the hour.

### Geo-replication vs geo-disaster recovery

| | Geo-disaster recovery | Geo-replication |
|---|---|---|
| Replicates | Metadata only (entities, config) | Metadata **and data** (messages, state changes, settlement) |
| Tier | Premium only | Premium only |
| Secondary | A second namespace, paired behind an **alias** FQDN; fail-forward only (pairing breaks on failover) | One secondary region (currently) behind the same namespace hostname; roles can swap back |
| Modes | n/a | **Synchronous** (RPO 0, higher latency, depends on both regions) or **asynchronous** with a configured maximum lag |
| Failover | Initiated by you | **Planned** promotion (waits for replication, namespace read-only meanwhile) or **forced** promotion (immediate, possible loss or duplicates) |
| Can combine? | | No, the two features can't be used together |

Microsoft recommends geo-replication for most disaster recovery scenarios. Current limitations include no large-message support. Promotion is always initiated by the customer:

```bash
az servicebus namespace failover --namespace-name <ns> -g <rg> --primary-location <new-primary-region>
```

### Sending and receiving (Python, keyless)

```python
from azure.identity import DefaultAzureCredential
from azure.servicebus import ServiceBusClient, ServiceBusMessage

NS = "sb-orders.servicebus.windows.net"
cred = DefaultAzureCredential()

with ServiceBusClient(NS, credential=cred) as client:
    with client.get_queue_sender("agent-tasks") as sender:
        sender.send_messages(ServiceBusMessage(
            '{"task":"summarise","docId":"D-42"}',
            message_id="D-42-summarise",        # enables duplicate detection
            session_id="conv-9f1",              # FIFO per conversation (session-enabled queue)
            application_properties={"priority": "high"},
        ))

    with client.get_queue_receiver("agent-tasks", session_id="conv-9f1", max_wait_time=5) as receiver:
        for msg in receiver:
            try:
                handle(str(msg))
                receiver.complete_message(msg)
            except TransientError:
                receiver.abandon_message(msg)          # retried; counts towards MaxDeliveryCount
            except PoisonError as e:
                receiver.dead_letter_message(msg, reason="Poison", error_description=str(e))
```

The identity needs **Azure Service Bus Data Sender** and **Azure Service Bus Data Receiver** (or **Data Owner**) at namespace or entity scope.

### Bicep: Premium namespace, session queue, local auth disabled

```bicep
resource sb 'Microsoft.ServiceBus/namespaces@2024-01-01' = {
  name: 'sb-orders-prod'
  location: location
  sku: { name: 'Premium', tier: 'Premium', capacity: 1 }
  properties: {
    disableLocalAuth: true          // Entra ID only, no SAS
    publicNetworkAccess: 'Disabled' // use a private endpoint
    minimumTlsVersion: '1.2'
  }
}

resource q 'Microsoft.ServiceBus/namespaces/queues@2024-01-01' = {
  parent: sb
  name: 'agent-tasks'
  properties: {
    requiresSession: true
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT1H'
    maxDeliveryCount: 5
    lockDuration: 'PT2M'
    deadLetteringOnMessageExpiration: true
    defaultMessageTimeToLive: 'P1D'
  }
}
```

Check the latest API version in the Bicep reference before you deploy. Some properties (`requiresSession`, `requiresDuplicateDetection`, partitioning) can only be set at creation.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Queue vs topic | Queue when one logical consumer type processes each message; topic when several independent subscribers each need a copy |
| Standard vs Premium | Premium for production workloads that need predictable latency, private networking, CMK, large messages or geo-replication. Standard for dev/test and modest, cost-sensitive workloads |
| Sessions | Use when ordering matters within a key. They limit parallelism to the number of active sessions, so choose a session key with enough cardinality |
| Large payloads | Even on Premium, prefer the claim check pattern (store in Blob, send a reference). Large messages reduce throughput and aren't yet supported with geo-replication |
| Lock duration vs renewal | Keep lock duration short (1–2 minutes) and renew automatically for long work; very long locks delay recovery when a worker dies |
| MaxDeliveryCount | Lower (3–5) for fast poison detection; higher when transient failures are common |
| Geo-replication mode | Synchronous when data loss is unacceptable and you can accept latency; asynchronous with a lag bound for most cases |

## Hands-on

1. Create a Standard namespace, a session-enabled queue and a topic with two subscriptions, one with a SQL filter `priority = 'high'`.
2. Assign yourself **Azure Service Bus Data Owner** on the namespace and run the Python sample with `DefaultAzureCredential`.
3. Send 10 messages across two session IDs and observe ordered processing per session.
4. Make the handler throw for one message and watch it reach the DLQ after `MaxDeliveryCount` attempts. Read the DLQ with `client.get_queue_receiver("agent-tasks", sub_queue=ServiceBusSubQueue.DEAD_LETTER)`.
5. Send the same `message_id` twice within the duplicate detection window and confirm only one is delivered.
6. Optional: create a Premium namespace and inspect the CPU and memory metrics.

## Security, cost and reliability notes

- **Security:** set `disableLocalAuth: true` and use Entra ID with RBAC. Scope roles to the entity where possible.
- **Security:** Premium supports private endpoints, IP firewall and customer-managed keys; Basic and Standard are reachable only over public endpoints (IP rules can be set via ARM, CLI or PowerShell).
- **Cost:** Standard is charged per operation, so polling receivers with short waits create cost. Use long-polling receive (`max_wait_time`) and batching.
- **Cost:** Premium is charged per MU-hour, regardless of traffic. Right-size with the CPU and memory metrics, and use autoscale.
- **Reliability:** consumers must be idempotent. Use `MessageId` plus duplicate detection, or a processed-message table (inbox pattern).
- **Reliability:** alert on DLQ depth (`DeadletteredMessages` metric) and active message count.
- **Reliability:** the SDKs retry transient errors. Configure retry options rather than writing your own loops.

## Common mistakes

- Using receive-and-delete for business messages, then losing them when a worker crashes.
- Never reading the dead-letter queue, so failures silently accumulate.
- Session keys with too few values (for example a country code), which serialise all processing.
- Long-running handlers without lock renewal, so messages are processed twice.
- Creating a new `ServiceBusClient` per message instead of reusing one connection.
- Expecting topics, sessions or transactions on the Basic tier.
- Trying to enable geo-replication and geo-disaster recovery on the same namespace.

## Check yourself

1. What happens to a message in peek-lock mode if the consumer crashes before completing it?
2. Name three reasons a message ends up in the dead-letter queue.
3. How do sessions guarantee ordering, and what do they cost you?
4. What does geo-replication replicate that geo-disaster recovery doesn't?
5. What is the difference between planned and forced promotion?
6. What is the maximum message size on Premium, and what protocol must you use?
7. Which tier do you need for private endpoints?
8. How do you prevent a duplicate send from creating two orders?

<details>
<summary>Answers</summary>

1. The lock expires and the message becomes available again; its delivery count increments.
2. Max delivery count exceeded, TTL expiry with dead-lettering on expiration enabled, filter evaluation error, or explicit dead-lettering by the application.
3. All messages with the same session ID are delivered in order to one receiver that holds the session lock. Parallelism is limited to the number of concurrently active sessions.
4. Message data and message state changes, in addition to metadata. Geo-DR replicates metadata only.
5. Planned waits for replication to catch up (namespace is read-only meanwhile) so there's no data loss; forced promotes immediately and can lose or duplicate unreplicated data.
6. 100 MB, over AMQP. HTTP is limited to 1 MB, and the per-entity default is 1 MB until you raise it.
7. Premium.
8. Set a deterministic `MessageId` and enable duplicate detection on the entity, and make the consumer idempotent.

</details>

## References

- [What is Azure Service Bus?](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview)
- [Premium messaging tier](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-premium-messaging)
- [Service Bus quotas](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-quotas)
- [Dead-letter queues](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-dead-letter-queues)
- [Message sessions](https://learn.microsoft.com/azure/service-bus-messaging/message-sessions)
- [Transactions](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-transactions)
- [Geo-replication](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-geo-replication)
- [Authenticate with Microsoft Entra ID](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-authentication-and-authorization)
- [Azure Service Bus client library for Python](https://learn.microsoft.com/python/api/overview/azure/servicebus-readme)
