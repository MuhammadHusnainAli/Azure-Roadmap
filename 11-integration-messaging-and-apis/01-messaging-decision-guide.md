# Messaging Decision Guide

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Azure has four messaging services that look similar from a distance: **Service Bus**, **Event Grid**, **Event Hubs** and **Storage Queues**. Picking the wrong one is one of the most common integration design mistakes. Teams build telemetry pipelines on Service Bus and hit throughput walls, or they put business orders on Event Grid and then can't guarantee ordering or transactions. AI workloads make the choice matter more. Ingestion pipelines, agent task queues, tool callbacks and model telemetry each need a different kind of messaging.

## Key concepts

### Events vs messages

The single most useful distinction, taken from Microsoft's own comparison guidance:

| | Event | Message |
|---|---|---|
| What it is | A lightweight notification that something happened | Raw data produced to be consumed by a specific component |
| Publisher's expectation | None about how it's handled | The consumer will process it, and the publisher may care about the outcome |
| Payload | Usually small; often a pointer to the data | The data itself (order, command, document reference) |
| Example | "Blob `invoice-42.pdf` was created" | "Process order 1042 for customer C-77" |

Events come in two flavours:

| Type | Meaning | Typical service |
|---|---|---|
| **Discrete event** | A single state change you react to | Event Grid |
| **Event series / stream** | A continuous flow of time-ordered data you analyse | Event Hubs |

### The four services in one line each

| Service | One-line description |
|---|---|
| **Azure Service Bus** | Enterprise message broker: queues and publish-subscribe topics with ordering (sessions), transactions, dead-lettering and duplicate detection |
| **Azure Event Grid** | Pub-sub event routing over HTTP (push or pull) and an MQTT broker for devices; reacts to discrete events from Azure services, your apps and partners |
| **Azure Event Hubs** | Big-data streaming platform: partitioned, retained log of events with Apache Kafka compatibility, Capture to storage and a schema registry |
| **Azure Queue Storage** | Simple, cheap, very large queues inside a storage account; at-least-once delivery with no broker features |

## How it works on Azure

### Feature comparison

| Capability | Service Bus | Event Grid | Event Hubs | Storage Queues |
|---|---|---|---|---|
| Model | Queue, pub-sub topic | Pub-sub (push or pull), MQTT | Partitioned log (stream) | Queue |
| Consumption | Pull (receive, peek-lock) | Push to handlers or pull (namespace topics) | Pull by consumer groups reading partitions | Pull (get, delete) |
| Ordering | FIFO per **session** | Not guaranteed | Ordered **within a partition** | Best effort, not guaranteed |
| Delivery | At-least-once (peek-lock) or at-most-once (receive and delete) | At-least-once, retries with backoff | Consumer tracks its own position (checkpoint) | At-least-once |
| Replay | No (message removed on completion) | No | Yes, within retention | No |
| Max message size | 256 KB (Basic/Standard), up to 100 MB (Premium, AMQP) | 1 MB per event (see current limits) | 256 KB (Basic), 1 MB (Standard/Premium), 20 MB (Dedicated) | 64 KB |
| Transactions | Yes (Standard and Premium) | No | No | No |
| Dead-lettering | Built in | Dead-letter to Blob storage (push) | No (consumer's job) | No (poison handling is your job) |
| Duplicate detection | Yes | No | No | No |
| Protocols | AMQP 1.0, HTTPS, JMS (Premium: JMS 2.0) | HTTPS, MQTT v3.1.1 and v5 | AMQP, Kafka, HTTPS (publish) | HTTPS |
| Typical throughput | Thousands of msgs/s per entity; scale with Premium messaging units | Millions of events/s across the service | Millions of events/s; scale with TUs, PUs or CUs | Up to 2,000 msgs/s per queue (1 KB) |

Numbers change. Check the quota pages in References before you size anything.

### Decision flow

```text
Is it telemetry or a high-volume stream you want to analyse, replay, or feed to Kafka clients?
  └─ Yes → Event Hubs
Is it a notification that "something happened" that one or many handlers react to?
  ├─ From Azure services (blob created, resource changed) or SaaS partners → Event Grid (system / partner topics)
  ├─ From IoT devices speaking MQTT → Event Grid namespace with MQTT broker
  └─ From your apps, fan-out to many subscribers, push to webhooks/functions → Event Grid
Is it a business command or transaction that must not be lost, must be processed once in order, or needs DLQ/transactions?
  └─ Yes → Service Bus (queue for one consumer type, topic for many)
Is it simple background work, huge backlog (>80 GB), lowest cost, no ordering needs?
  └─ Yes → Storage Queues
```

### Combining services

Real architectures usually combine them:

| Pattern | Services |
|---|---|
| Blob upload triggers RAG ingestion | Event Grid (BlobCreated) → Service Bus queue (buffer, DLQ) → Functions / Container Apps workers |
| IoT telemetry with alerting | Event Grid MQTT broker → Event Hubs → Stream Analytics / Fabric Eventstream; anomalies → Event Grid → Logic Apps |
| Order processing with audit stream | Service Bus topic (orders) for processing; same events mirrored to Event Hubs for analytics |
| Model and agent telemetry | Application/API Management logs → Event Hubs → Fabric Real-Time Intelligence or Azure Data Explorer |
| Agent task queue | Service Bus queue with sessions (one session per conversation) → agent workers |

### AI workload mapping

| AI scenario | Recommended | Why |
|---|---|---|
| Long-running agent tasks requested by a front end | Service Bus queue | Durable, DLQ, retries, sessions per conversation |
| Document ingestion for RAG at scale | Event Grid → Service Bus | Event Grid detects new blobs; Service Bus buffers and smooths load to your embedding quota |
| Streaming prompts, completions and token usage for analytics | Event Hubs | High volume, retention, Kafka and Fabric integration |
| Triggering re-indexing when a document changes | Event Grid | Discrete change notification |
| Batch scoring jobs on a budget | Storage Queues | Cheap, simple, huge backlog |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Service Bus vs Storage Queues | Service Bus when you need ordering, sessions, transactions, DLQ, duplicate detection, topics, or messages up to 100 MB. Storage Queues when you need a backlog over 80 GB, a simple pull queue, or the lowest cost |
| Event Grid vs Service Bus topics | Event Grid for reactive, lightweight notifications with push delivery and many event sources. Service Bus topics when subscribers must process business messages reliably with filters, sessions and transactions |
| Event Hubs vs Service Bus | Event Hubs for streams (many events, replay, partitions, analytics). Service Bus for discrete business messages that each need individual handling and settlement |
| Event Hubs vs Kafka on VMs or AKS | Event Hubs gives Kafka-protocol compatibility as a managed service. Self-managed Kafka only when you need Kafka features Event Hubs doesn't support |
| Push vs pull | Push (Event Grid) is simplest when the handler has an endpoint. Pull (Service Bus, Event Hubs, Event Grid namespace pull) when the consumer controls pace, sits behind a private network, or must not expose an endpoint |

## Hands-on

Create one of each and send a test message to feel the differences:

```bash
RG=rg-msg-lab; LOC=swedencentral; SUF=$RANDOM
az group create -n $RG -l $LOC

# Service Bus queue
az servicebus namespace create -g $RG -n sb-lab-$SUF --sku Standard
az servicebus queue create -g $RG --namespace-name sb-lab-$SUF -n orders

# Event Hubs
az eventhubs namespace create -g $RG -n eh-lab-$SUF --sku Standard
az eventhubs eventhub create -g $RG --namespace-name eh-lab-$SUF -n telemetry --partition-count 4

# Event Grid custom topic
az eventgrid topic create -g $RG -n egt-lab-$SUF -l $LOC

# Storage queue
az storage account create -g $RG -n stlab$SUF --sku Standard_LRS
az storage queue create --account-name stlab$SUF -n jobs --auth-mode login
```

Then, for each, answer: how do I read the message? Can I read it twice? What happens if my consumer crashes mid-processing?

## Security, cost and reliability notes

- **Security:** all four support Microsoft Entra ID authentication and Azure RBAC data-plane roles (for example *Azure Service Bus Data Sender*, *Azure Event Hubs Data Receiver*, *EventGrid Data Sender*, *Storage Queue Data Contributor*). Prefer managed identities over SAS keys or connection strings, and disable local (key) auth where the service supports it.
- **Security:** private endpoints need specific tiers: Service Bus **Premium**, Event Hubs **Standard and above**, Event Grid custom topics and namespaces (pull delivery only for private consumption).
- **Cost:** Service Bus Standard and Event Grid bill per operation; Service Bus Premium and Event Hubs Premium/Dedicated bill for reserved capacity. A chatty design on a per-operation tier can cost more than a reserved tier.
- **Reliability:** all are at-least-once. Make consumers **idempotent**.
- **Reliability:** for regional resilience, Service Bus Premium and Event Hubs Premium/Dedicated support geo-replication of data. Event Grid and Storage have their own disaster recovery models; see each service's file.

## Common mistakes

- Using Event Grid for business commands that must be processed once and in order.
- Using Service Bus for high-volume telemetry that needs replay and analytics.
- Assuming Storage Queues guarantee FIFO ordering.
- Sending large payloads through a broker instead of a reference to blob storage (see the claim check pattern in file 10).
- Forgetting that every service redelivers: non-idempotent consumers create duplicate orders.
- Choosing by familiarity ("we know Kafka") rather than by message semantics.

## Check yourself

1. What is the difference between an event and a message?
2. Which service would you choose for IoT devices that speak MQTT?
3. Which services guarantee ordering, and at what scope?
4. A team needs transactions across sending two messages. Which service?
5. When would Storage Queues be a better choice than Service Bus?
6. Which service lets a consumer re-read events from yesterday?
7. Design a messaging chain for "a PDF lands in Blob storage and must be chunked, embedded and indexed".

<details>
<summary>Answers</summary>

1. An event is a lightweight notification of a state change; the publisher has no expectation about handling. A message carries data that the publisher expects a consumer to process.
2. Event Grid namespaces with the MQTT broker feature (MQTT v3.1.1 and v5).
3. Service Bus guarantees FIFO within a session; Event Hubs guarantees order within a partition. Event Grid and Storage Queues don't guarantee ordering.
4. Service Bus (Standard or Premium), which supports transactions of up to 100 messages.
5. When you need a very large backlog (over 80 GB), only simple at-least-once queueing, the lowest cost, or you're already using a storage account and don't need broker features.
6. Event Hubs, within the retention period of the tier (up to 90 days on Premium and Dedicated).
7. Blob storage raises `BlobCreated` → Event Grid subscription delivers to a Service Bus queue → a worker (Functions or Container Apps) reads, extracts, chunks, embeds and indexes; failures go to the dead-letter queue with alerts.

</details>

## References

- [Choose between Azure messaging services](https://learn.microsoft.com/azure/service-bus-messaging/compare-messaging-services)
- [Storage queues and Service Bus queues compared](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-azure-and-storage-queues-compared-contrasted)
- [Service Bus quotas](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-quotas)
- [Event Hubs quotas](https://learn.microsoft.com/azure/event-hubs/event-hubs-quotas)
- [Event Grid quotas and limits](https://learn.microsoft.com/azure/event-grid/quotas-limits)
- [Asynchronous messaging options (Architecture Center)](https://learn.microsoft.com/azure/architecture/guide/technology-choices/messaging)
