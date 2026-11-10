# Azure Event Hubs

## Why it matters

Event Hubs is Azure's managed event streaming platform: a partitioned, durable, append-only log that can take in millions of events per second and let many independent consumers read them at their own pace. It speaks the **Apache Kafka** protocol, so Kafka clients work unchanged. It can **Capture** streams to storage automatically, and it has a **schema registry**. For AI and data teams it's the front door for telemetry, clickstreams, logs, model and agent traces, and real-time analytics in Fabric, Stream Analytics and Azure Data Explorer.

## Key concepts

### The log model

```text
Producers ──► Event hub "telemetry" (4 partitions)
               P0: e0 e4 e8 ...   ◄── consumer group "analytics"  (reader per partition, own offset)
               P1: e1 e5 e9 ...   ◄── consumer group "alerts"     (independent offsets)
               P2: ...
               P3: ...
Events stay until retention expires, regardless of whether they were read.
```

| Term | Meaning |
|---|---|
| **Namespace** | Container and endpoint (`<name>.servicebus.windows.net`); unit of tier and capacity |
| **Event hub** | The stream (equivalent to a Kafka **topic**) |
| **Partition** | An ordered sequence of events; the unit of parallelism and ordering |
| **Partition key** | Value hashed to choose a partition, so related events stay in order (for example a device ID) |
| **Consumer group** | A view of the whole stream with its own read positions; each downstream application gets its own |
| **Offset / sequence number** | Position of an event in a partition |
| **Checkpoint** | A consumer's saved position (stored by the SDK in Blob storage) so it can resume |
| **Retention** | How long events are kept, from 1 day (Basic) to 90 days (Premium, Dedicated) |

Ordering is guaranteed **within a partition only**. Event Hubs doesn't delete events when they're read, and consumers don't "complete" events as they do in Service Bus.

### Capacity units by tier

| Tier | Unit | What one unit gives |
|---|---|---|
| Basic, Standard | **Throughput unit (TU)** | Ingress up to 1 MB/s or 1,000 events/s; egress up to 2 MB/s or 4,096 events/s. Up to 40 TUs; **auto-inflate** can raise TUs automatically |
| Premium | **Processing unit (PU)** | Isolated CPU and memory; throughput depends on workload. 1, 2, 4, 8 or 16 PUs |
| Dedicated | **Capacity unit (CU)** | Single-tenant cluster; scale in CUs (more than 10 by support request) |

### Tier comparison (checked 2026-11)

| | Basic | Standard | Premium | Dedicated |
|---|---|---|---|---|
| Max event (publication) size | 256 KB | 1 MB | 1 MB | 20 MB |
| Max retention | 1 day | 7 days | 90 days | 90 days |
| Partitions per event hub | 32 | 32 | 100 (200 per PU at namespace level) | 1,024 (2,000 per CU) |
| Consumer groups per event hub | 1 | 20 | 100 | 1,000 |
| Event hubs per namespace | 10 | 10 | 100 per PU | 1,000 |
| Kafka endpoint | No | Yes | Yes | Yes |
| Capture | No | Yes, billed per hour | Included | Included |
| Schema registry | No | Yes (25 MB) | Yes (100 MB) | Yes (1 GB) |
| Private link, IP firewall | No | Yes | Yes | Yes |
| Customer-managed keys | No | No | Yes | Yes |
| Dynamic partition scale-out | No | No | Yes | Yes |
| Log compaction | No | Yes | Yes | Yes |
| Geo-disaster recovery (metadata) | No | Yes | Yes | Yes |
| Geo-replication (data) | No | No | Yes | Yes |

## How it works on Azure

### Kafka endpoint

Event Hubs exposes a Kafka-compatible endpoint on port **9093** (`<namespace>.servicebus.windows.net:9093`) for Standard, Premium and Dedicated. Existing Kafka producers and consumers, Kafka Connect and MirrorMaker work with a configuration change. You don't run brokers or ZooKeeper/KRaft. Authentication uses SASL with OAuth (Entra ID, recommended) or SASL PLAIN with a connection string.

```properties
bootstrap.servers=evh-telemetry.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
# configure an OAuth callback handler that obtains Entra ID tokens
# (scope: https://eventhubs.azure.net/.default)
```

| Kafka concept | Event Hubs equivalent |
|---|---|
| Cluster | Namespace |
| Topic | Event hub |
| Partition | Partition |
| Consumer group | Consumer group (Kafka consumer groups are created automatically) |
| Offset | Offset |

Some Kafka features differ or aren't supported (for example Kafka transactions and certain admin APIs vary by tier). Check the Kafka feature matrix before migrating.

### Capture

**Capture** writes the stream automatically to **Azure Blob Storage** or **Data Lake Storage Gen2** in **Avro** or **Parquet** format (Parquet output is configured through the no-code editor with Stream Analytics). You set a time window and a size window, and whichever is reached first triggers a file. It's ideal for building a raw (bronze) layer and replaying history into model training or analytics.

### Schema registry

The schema registry stores **Avro** and **JSON Schema** (and Protobuf, check status) schemas in **schema groups** with a compatibility policy (none, backward, forward). Producers serialise with a schema ID rather than embedding the full schema, and consumers fetch and cache it. It works with both the Event Hubs SDKs and Kafka clients.

### Produce and consume (Python, keyless)

```python
from azure.identity import DefaultAzureCredential
from azure.eventhub import EventHubProducerClient, EventHubConsumerClient, EventData
from azure.eventhub.extensions.checkpointstoreblob import BlobCheckpointStore

NS = "evh-telemetry.servicebus.windows.net"
cred = DefaultAzureCredential()

producer = EventHubProducerClient(fully_qualified_namespace=NS, eventhub_name="llm-usage", credential=cred)
with producer:
    batch = producer.create_batch(partition_key="app-chat")   # keep one app's events in order
    batch.add(EventData('{"app":"chat","model":"gpt-x","promptTokens":812,"completionTokens":143}'))
    producer.send_batch(batch)

checkpoints = BlobCheckpointStore(
    blob_account_url="https://stcheckpoints.blob.core.windows.net",
    container_name="eh-checkpoints", credential=cred)

consumer = EventHubConsumerClient(
    fully_qualified_namespace=NS, eventhub_name="llm-usage",
    consumer_group="cost-analytics", credential=cred, checkpoint_store=checkpoints)

def on_event(partition_context, event):
    record(event.body_as_json())
    partition_context.update_checkpoint(event)   # checkpoint periodically in production, not every event

with consumer:
    consumer.receive(on_event=on_event, starting_position="-1")   # "-1" = from the beginning
```

Roles: **Azure Event Hubs Data Sender** and **Azure Event Hubs Data Receiver**; the consumer also needs **Storage Blob Data Contributor** on the checkpoint container.

### Bicep

```bicep
resource ns 'Microsoft.EventHub/namespaces@2024-01-01' = {
  name: 'evh-telemetry-prod'
  location: location
  sku: { name: 'Standard', tier: 'Standard', capacity: 2 }
  properties: {
    isAutoInflateEnabled: true
    maximumThroughputUnits: 10
    disableLocalAuth: true
    minimumTlsVersion: '1.2'
  }
}

resource hub 'Microsoft.EventHub/namespaces/eventhubs@2024-01-01' = {
  parent: ns
  name: 'llm-usage'
  properties: {
    partitionCount: 8
    retentionDescription: { cleanupPolicy: 'Delete', retentionTimeInHours: 168 }
  }
}

resource cg 'Microsoft.EventHub/namespaces/eventhubs/consumergroups@2024-01-01' = {
  parent: hub
  name: 'cost-analytics'
}
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Partition count | Set it for **peak consumer parallelism**, not just ingress. In Basic and Standard you can't change it after creation; Premium and Dedicated can increase it (but adding partitions changes key-to-partition mapping) |
| Partition key vs round robin | Use a key when per-entity order matters; round robin (no key) for best distribution and availability |
| Tier | Standard for most workloads; Premium for isolation, longer retention, CMK and predictable latency; Dedicated for very large, single-tenant scale or 20 MB events |
| Event Hubs vs Service Bus | Streams and replay → Event Hubs. Per-message settlement, DLQ, sessions, transactions → Service Bus |
| Kafka on Event Hubs vs self-managed or Confluent | Event Hubs for managed simplicity in Azure; self-managed or Confluent Cloud for full Kafka API fidelity (Streams, transactions, ecosystem features) |
| Capture vs consumer that writes files | Capture is zero-code and reliable for raw archival; a custom consumer when you need transformation |
| Fabric Eventstream vs direct consumers | If the analytics destination is Fabric, an Eventstream can read from Event Hubs without code (topic 12) |

## Hands-on

1. Create a Standard namespace with an 8-partition event hub and two consumer groups, then disable local auth.
2. Assign yourself Data Sender and Data Receiver, then run the producer with two partition keys and inspect `partition_context.partition_id` in the consumer.
3. Stop the consumer, produce 100 events, restart, and confirm it resumes from the checkpoint.
4. Run a second consumer group from `"-1"` and confirm it reads the full history independently.
5. Enable Capture to an ADLS Gen2 container with a 5-minute window and inspect the Avro files.
6. Optional: point a Kafka client (for example `confluent-kafka` in Python) at port 9093 with OAuth.

## Security, cost and reliability notes

- **Security:** set `disableLocalAuth` and use Entra ID with RBAC for both the SDKs and Kafka (OAUTHBEARER).
- **Security:** private endpoints are available from Standard upwards. CMK requires Premium or Dedicated.
- **Cost:** Standard bills TUs per hour plus ingress events; Capture is extra on Standard. Auto-inflate scales TUs up but **not down**, so reduce them yourself or with automation.
- **Cost:** retention beyond the default and large numbers of consumer groups don't usually cost extra, but storage for Capture and checkpoints does.
- **Reliability:** checkpoint deliberately. Checkpointing every event is slow; checkpointing rarely means more reprocessing after a restart. Consumers must handle duplicates.
- **Reliability:** consider **geo-replication** (Premium and Dedicated) for data replication across regions, or geo-disaster recovery for metadata-only failover.
- **Reliability:** watch `ThrottledRequests`, `IncomingMessages` vs `OutgoingMessages`, and consumer lag.

## Common mistakes

- Too few partitions, so consumers can't scale out, and the count can't be changed on Standard.
- Using one consumer group for several applications, so they steal partitions from each other.
- Expecting Service Bus semantics (DLQ, per-message completion).
- Forcing one hot partition key for most traffic, which creates a bottleneck.
- Forgetting the checkpoint store, so every restart replays the entire stream.
- Leaving auto-inflated TUs high after a spike.

## Check yourself

1. What determines ordering in Event Hubs?
2. Why should each downstream application have its own consumer group?
3. What does a throughput unit provide?
4. What is Capture, and what formats does it write?
5. Which tiers support the Kafka endpoint, and on which port?
6. How long can you retain events on Premium?
7. A team needs per-message dead-lettering. Is Event Hubs right?
8. Can you change the partition count of a Standard event hub after creation?

<details>
<summary>Answers</summary>

1. Events are ordered within a partition; use a partition key to keep related events in the same partition.
2. Each consumer group tracks its own offsets and readers, so applications read the stream independently without interfering.
3. Up to 1 MB/s or 1,000 events/s ingress and 2 MB/s or 4,096 events/s egress (Basic and Standard).
4. Automatic, zero-code archival of the stream to Blob Storage or ADLS Gen2, in Avro (and Parquet via the no-code editor).
5. Standard, Premium and Dedicated, on port 9093.
6. Up to 90 days.
7. No. Event Hubs has no per-message settlement or DLQ; use Service Bus, or handle failures in the consumer.
8. No. In Basic and Standard the partition count is fixed at creation; Premium and Dedicated support increasing it.

</details>

## References

- [What is Azure Event Hubs?](https://learn.microsoft.com/azure/event-hubs/event-hubs-about)
- [Event Hubs features and terminology](https://learn.microsoft.com/azure/event-hubs/event-hubs-features)
- [Event Hubs quotas and limits](https://learn.microsoft.com/azure/event-hubs/event-hubs-quotas)
- [Event Hubs for Apache Kafka](https://learn.microsoft.com/azure/event-hubs/azure-event-hubs-apache-kafka-overview)
- [Capture events](https://learn.microsoft.com/azure/event-hubs/event-hubs-capture-overview)
- [Schema Registry](https://learn.microsoft.com/azure/event-hubs/schema-registry-overview)
- [Scalability: throughput and processing units](https://learn.microsoft.com/azure/event-hubs/event-hubs-scalability)
- [Geo-replication](https://learn.microsoft.com/azure/event-hubs/geo-replication)
- [Azure Event Hubs client library for Python](https://learn.microsoft.com/python/api/overview/azure/eventhub-readme)
