# Azure Event Grid

> Topic: 11 · Integration, Messaging and APIs · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Event Grid is how Azure tells your code that something happened: a blob was uploaded, a resource was created, a key is about to expire, a device published a reading. It's a fully managed publish-subscribe service that routes **discrete events** to handlers by push, lets consumers **pull** events from namespace topics, and runs a cloud **MQTT broker** for IoT clients. Most event-driven AI pipelines start with Event Grid, for example "a new document arrived, so re-index it".

## Key concepts

### Two resource models

Event Grid has two families of resources. Knowing which one you are using explains most of the differences in behaviour.

| | Event Grid basic (custom, system, partner topics, domains) | Event Grid **namespaces** |
|---|---|---|
| Resource types | Custom topic, system topic, partner topic, domain | Namespace containing **namespace topics**, MQTT clients, topic spaces |
| Delivery | **Push** only | **Push** (to Event Hubs or webhooks) and **pull** |
| Event schemas | Event Grid schema, CloudEvents 1.0, custom input schema | CloudEvents 1.0 |
| MQTT broker | No | Yes |
| Azure service (system) events | Yes, via system topics | Not directly; route them in through a custom setup |
| Typical use | Reacting to Azure resource events; simple app pub-sub to webhooks, Functions, queues | High-scale app pub-sub with pull consumers, private consumption, IoT over MQTT |

### Core terms

| Term | Meaning |
|---|---|
| **Event** | A small record (≤1 MB) describing what happened: source, type, subject, time, ID, data |
| **Publisher / source** | The system that emits the event (Azure Storage, your app, a SaaS partner) |
| **Topic** | The endpoint publishers send to |
| **System topic** | A topic representing an Azure service's built-in events (for example a storage account) |
| **Partner topic** | Events from a third-party SaaS provider, such as Auth0 or SAP (check the current partner list) |
| **Domain** | A management container for thousands of related topics, for example one per tenant |
| **Event subscription** | Defines which events (filters) go to which handler, with retry and dead-letter settings |
| **Event handler** | The destination: webhook, Azure Function, Service Bus, Event Hubs, Storage queue, Relay hybrid connection |

### CloudEvents

Event Grid supports the **CNCF CloudEvents 1.0** specification, a vendor-neutral event envelope. Use it for new designs because it's portable and is the only schema namespaces accept.

```json
{
  "specversion": "1.0",
  "type": "Contoso.Documents.DocumentUploaded",
  "source": "/tenants/contoso/apps/intake",
  "subject": "documents/invoice-42.pdf",
  "id": "8b0b4c1f-7e3d-4a1b-9f60-1b2f5c0c1a11",
  "time": "2026-11-05T09:30:00Z",
  "datacontenttype": "application/json",
  "data": { "blobUrl": "https://stdocs.blob.core.windows.net/in/invoice-42.pdf", "sizeBytes": 183422 }
}
```

### Push vs pull delivery

| | Push | Pull (namespace topics only) |
|---|---|---|
| Who initiates | Event Grid calls your handler | Your consumer connects and receives a batch |
| Handler needs | A reachable endpoint (or Azure service handler) | Outbound access to Event Grid only |
| Private networking | Handler must be reachable from Event Grid | Consume over **private endpoints** |
| Control of pace | Event Grid retries with exponential backoff for up to **24 hours** (default) | Consumer controls when and how much to read; can **release** events for later redelivery, **reject** or **acknowledge** |
| Settlement | HTTP response code | Explicit acknowledge, release, reject, renew lock |

Microsoft's guidance: choose **pull** when you need control over when to consume, private-link consumption, or you can't expose an endpoint. Choose **push** when you want to avoid polling and your handler can receive calls.

### MQTT broker

Event Grid namespaces include an MQTT broker (GA) for device and application messaging:

| Feature | Detail |
|---|---|
| Protocols | MQTT v3.1.1 and v5.0, each also over WebSockets; TLS 1.2 and 1.3 |
| Authentication | X.509 certificates, Microsoft Entra ID tokens, OAuth 2.0 JWT, custom webhook authentication |
| Authorisation | **Client groups** and **topic spaces** with permission bindings; topic templates with variables for per-client access |
| Messaging | One-to-many, many-to-one, one-to-one; retained messages; shared subscriptions; MQTT 5 features such as subscription identifiers |
| Routing | Route MQTT messages to a namespace topic or custom topic, then on to Event Hubs, Functions, webhooks and other handlers |
| HTTP publish | Publish MQTT messages over HTTPS (Preview, checked 2026-11) |
| Edge | Bridges to the MQTT broker in Azure IoT Operations |

## How it works on Azure

### React to blob uploads (system topic → Function)

```bash
RG=rg-eg-lab; ST=stdocs$RANDOM; LOC=swedencentral
az storage account create -g $RG -n $ST -l $LOC --sku Standard_LRS --kind StorageV2

STID=$(az storage account show -g $RG -n $ST --query id -o tsv)
FUNCID=$(az functionapp function show -g $RG -n func-ingest --function-name OnBlob --query id -o tsv)

az eventgrid event-subscription create \
  --name blob-to-ingest \
  --source-resource-id $STID \
  --endpoint-type azurefunction \
  --endpoint $FUNCID \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/in/ \
  --event-delivery-schema cloudeventschemav1_0 \
  --deadletter-endpoint "$STID/blobServices/default/containers/deadletter" \
  --max-delivery-attempts 10 --event-ttl 1440
```

Filtering options: event types, **subject begins with / ends with**, and **advanced filters** on data fields (string, number, boolean operators).

### Publish and pull with a namespace topic (Python, keyless)

```python
from azure.identity import DefaultAzureCredential
from azure.eventgrid import EventGridPublisherClient, EventGridConsumerClient
from azure.core.messaging import CloudEvent

ENDPOINT = "https://<namespace>.<region>-1.eventgrid.azure.net"
cred = DefaultAzureCredential()

publisher = EventGridPublisherClient(ENDPOINT, cred, namespace_topic="documents")
publisher.send([CloudEvent(source="/apps/intake", type="Contoso.DocumentUploaded",
                           data={"docId": "D-42"})])

consumer = EventGridConsumerClient(ENDPOINT, cred, namespace_topic="documents",
                                   subscription="indexer")
details = consumer.receive(max_events=10, max_wait_time=10)
tokens = []
for d in details:
    process(d.event)
    tokens.append(d.broker_properties.lock_token)
if tokens:
    consumer.acknowledge(lock_tokens=tokens)
```

The publisher needs **EventGrid Data Sender**; the consumer needs **EventGrid Data Receiver**. Class and method names follow the current `azure-eventgrid` package (v4.20+); check the SDK reference if they have moved.

### Delivery, retry and dead-lettering (push)

| Setting | Default and behaviour |
|---|---|
| Retry | Exponential backoff with jitter; default **30 attempts** within **24 hours** (both configurable down) |
| Non-retried errors | Some 4xx responses (for example 400, 413) aren't retried and go straight to dead-letter if configured |
| Dead-letter destination | A Blob storage container |
| Batching | Configurable max events per batch and preferred batch size |
| Delivery identity | Event Grid can use a **managed identity** to deliver to Service Bus, Event Hubs, Storage queues and dead-letter storage |

### Webhook validation and security

Webhook handlers must prove ownership by answering a **validation handshake** (`SubscriptionValidationEvent` or the CloudEvents `OPTIONS` abuse-protection handshake). Protect webhooks with **Microsoft Entra ID** authentication on delivery, where Event Grid sends a bearer token your endpoint validates. Azure Functions and Service Bus handlers don't need the handshake.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Basic topics vs namespaces | System events from Azure services → system topics. New app pub-sub needing pull, private consumption or MQTT → namespaces. Simple push to Functions or webhooks → custom topics are fine |
| Push directly to a function vs via a queue | Push to Service Bus or a Storage queue first when processing is slow, rate-limited (embedding quotas) or must survive downstream outages |
| Event Grid schema vs CloudEvents | CloudEvents for new designs and anything that may move between platforms |
| Event Grid vs Event Hubs | Event Grid for discrete, reactive events; Event Hubs for continuous high-volume streams and replay |
| MQTT broker vs IoT Hub | Event Grid MQTT for standards-based pub-sub messaging at scale and flexible topic hierarchies; IoT Hub for device management features (twins, direct methods, device provisioning). See topic 32 |

## Hands-on

1. Create a storage account and a Function with an Event Grid trigger. Subscribe to `BlobCreated` for one container using CloudEvents schema.
2. Upload a file and check the function log. Delete the function temporarily, upload again, and confirm the event appears in the dead-letter container after retries.
3. Create an Event Grid namespace and namespace topic with a **queue** (pull) event subscription. Assign yourself **EventGrid Data Sender** and **EventGrid Data Receiver**, then run the Python publisher and consumer.
4. Optional: enable the MQTT broker on the namespace, register a client with an X.509 thumbprint, create a topic space `devices/+/telemetry`, and publish with `mosquitto_pub`.

## Security, cost and reliability notes

- **Security:** prefer Entra ID for publishers and disable local (access key / SAS) authentication on topics and namespaces when possible.
- **Security:** private endpoints are available for custom topics, domains and namespaces. For push delivery to private handlers, deliver to Azure services (Service Bus, Event Hubs, Storage) using managed identity, or use pull delivery.
- **Security:** validate the issuer, audience and signature on webhook tokens; never rely on obscurity of the URL.
- **Cost:** basic topics bill per operation (publishes, deliveries, advanced filter matches). Namespaces bill **throughput units** plus operations; MQTT has its own meters. Check the pricing page.
- **Reliability:** delivery is at-least-once and **not ordered**. Make handlers idempotent using the event `id`.
- **Reliability:** always configure dead-lettering and alert on `DeadLetteredCount` and `DroppedEventCount`.
- **Reliability:** zone redundancy and regional disaster recovery differ between basic topics and namespaces, and between regions with and without pairs. Read the Event Grid reliability guide for your resource type before you promise an RTO. For critical flows, publish to topics in two regions.

## Common mistakes

- Assuming events arrive in order or exactly once.
- Pushing directly into a slow handler that then times out, causing retries and duplicates.
- Forgetting the webhook validation handshake, so the subscription never activates.
- No dead-letter destination, so failed events disappear after retries.
- Using Event Grid for large payloads instead of including a reference to the data.
- Trying to receive Azure system events on a namespace topic directly.

## Check yourself

1. What are the two Event Grid resource models, and which supports pull delivery?
2. When should you choose pull over push?
3. What does CloudEvents give you?
4. Which MQTT versions does the Event Grid broker support?
5. What happens to an event that can't be delivered after retries?
6. Why might you put Service Bus between Event Grid and an ingestion worker?
7. What is the default retry window for push delivery?

<details>
<summary>Answers</summary>

1. Event Grid basic (custom, system and partner topics, and domains), which is push only, and Event Grid namespaces, which support push and pull for namespace topics plus the MQTT broker.
2. When the consumer must control its pace, must consume over a private endpoint, can't expose an endpoint, or needs to release events back for later.
3. A vendor-neutral, standard event envelope (CNCF CloudEvents 1.0) that is portable across platforms and tools.
4. MQTT v3.1.1 and v5.0, each also over WebSockets.
5. It's dropped, or written to the configured dead-letter Blob container.
6. To buffer bursts, smooth load to rate-limited back ends such as embedding quotas, add sessions or ordering, and get broker dead-lettering and retries independent of Event Grid's retry window.
7. 24 hours (with up to 30 attempts by default).

</details>

## References

- [What is Azure Event Grid?](https://learn.microsoft.com/azure/event-grid/overview)
- [Event Grid namespaces concepts](https://learn.microsoft.com/azure/event-grid/concepts-event-grid-namespaces)
- [Pull delivery overview](https://learn.microsoft.com/azure/event-grid/pull-delivery-overview)
- [Push delivery overview](https://learn.microsoft.com/azure/event-grid/push-delivery-overview)
- [MQTT broker overview](https://learn.microsoft.com/azure/event-grid/mqtt-overview)
- [CloudEvents v1.0 schema with Event Grid](https://learn.microsoft.com/azure/event-grid/cloud-event-schema)
- [Delivery and retry](https://learn.microsoft.com/azure/event-grid/delivery-and-retry)
- [Event Grid quotas and limits](https://learn.microsoft.com/azure/event-grid/quotas-limits)
- [Azure Event Grid client library for Python](https://learn.microsoft.com/python/api/overview/azure/eventgrid-readme)
