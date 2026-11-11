# Azure IoT Hub and Device Provisioning

## Why it matters

Connected devices (sensors, meters, vehicles, medical devices, machines) generate the operational data that analytics and AI depend on, but connecting millions of constrained devices securely is hard: each needs an identity, a secure channel, a way to receive commands and configuration, and a way to be provisioned without someone typing credentials into it in a factory. **Azure IoT Hub** is the managed cloud gateway for device-to-cloud and cloud-to-device communication at scale, and the **Device Provisioning Service (DPS)** registers devices automatically into the right hub with the right identity. These remain the foundation for cloud-connected device solutions, alongside Azure IoT Operations for industrial edge scenarios (file 02).

## Key concepts

### IoT Hub capabilities

| Capability | Detail |
|---|---|
| **Device identity registry** | Per-device identity with symmetric keys, X.509 certificates (CA-signed or self-signed thumbprints) |
| **Device-to-cloud (D2C) telemetry** | Millions of devices, millions of events per second; built-in Event Hubs-compatible endpoint |
| **Message routing** | Route by properties/body to Event Hubs, Service Bus, Storage, Cosmos DB; enrichments |
| **Cloud-to-device (C2D)** | Messages, **direct methods** (request-reply), **device twins** (desired/reported properties) |
| **File upload** | Devices upload files to linked Storage |
| **Device twins and module twins** | JSON documents for state, configuration and metadata; queryable |
| **Automatic device management** | Configurations targeting device groups by twin tags |
| **Protocols** | MQTT, AMQP, HTTPS (and over WebSockets) |
| **Monitoring** | Azure Monitor metrics/logs, Event Grid events (device connected/disconnected/created) |
| **Tiers** | Basic (D2C only) and Standard (full features), units per tier for throughput |

### Device Provisioning Service

| Concept | Detail |
|---|---|
| **Zero-touch provisioning** | Devices contact a global DPS endpoint at first boot, authenticate and get assigned to a hub |
| **Attestation** | X.509 certificates, TPM, symmetric keys |
| **Enrolments** | Individual or group enrolments (group by CA certificate or key derivation) |
| **Allocation policies** | Evenly weighted, lowest latency, static, or custom (Azure Function) across multiple hubs |
| **Reprovisioning** | Move devices between hubs; reset identities |

### Security essentials

Use X.509 or TPM attestation in production; unique identities per device; certificate rotation; least-privilege shared access policies for back-end services (or Entra RBAC for service access); private endpoints and IP filters; Defender for IoT for device and network threat detection.

## How it works on Azure

Create a hub, DPS and link them:

```bash
az iot hub create -g rg-iot -n iothub-contoso-weu --sku S1 --partition-count 4
az iot dps create -g rg-iot -n dps-contoso
az iot dps linked-hub create -g rg-iot --dps-name dps-contoso --hub-name iothub-contoso-weu
az iot dps enrollment-group create -g rg-iot --dps-name dps-contoso --enrollment-id meters-x509 \
  --certificate-path ./intermediate-ca.pem
```

Route telemetry to Event Hubs for stream processing and to Storage for archive:

```bash
az iot hub message-endpoint create eventhub -g rg-iot --hub-name iothub-contoso-weu -n eh-telemetry \
  --connection-string "<event hub connection or use identity-based endpoint>"
az iot hub message-route create -g rg-iot --hub-name iothub-contoso-weu --route-name hot-path \
  --endpoint-name eh-telemetry --source DeviceMessages --condition "messageType = 'telemetry'"
```

Device twin desired property update for a firmware setting:

```bash
az iot hub device-twin update -n iothub-contoso-weu -d meter-0042 --desired '{"samplingIntervalSec": 30}'
```

Typical architecture: devices → DPS → IoT Hub → routing → Event Hubs → Stream Analytics/Fabric Real-Time Intelligence → dashboards, alerts, Digital Twins (file 03), storage for ML.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **IoT Hub vs IoT Operations vs Event Grid MQTT broker** | IoT Hub for cloud-connected device fleets; IoT Operations for industrial edge data planes; Event Grid MQTT broker for MQTT pub/sub scenarios in the cloud. |
| **Attestation** | X.509 with a PKI or TPM for production; symmetric keys for prototypes. |
| **Hub topology** | Multiple hubs per region/tenant for scale and isolation, allocated by DPS. |
| **Throughput** | Size units and partitions for peak message rates; batch telemetry. |
| **Data path** | Hot path (streaming) and cold path (storage) separation. |

## Hands-on

1. Create IoT Hub and DPS; provision a simulated device with an X.509 group enrolment.
2. Send telemetry and route it to Event Hubs and Storage.
3. Update a desired property and handle it on the device.
4. Invoke a direct method on the device.
5. Subscribe to device connected/disconnected events with Event Grid.

## Security, cost and reliability notes

- **Per-device identity** and certificate lifecycle management are essential.
- **Cost**: tier and units; messages counted in 4 KB chunks.
- **Reliability**: devices must buffer and retry; consider regional failover (manual failover for IoT Hub).
- **Firmware**: secure update mechanisms (Device Update for IoT Hub where applicable).

## Common mistakes

| Mistake | Fix |
|---|---|
| Shared keys across devices | Per-device identity |
| Provisioning manually | DPS |
| Basic tier when C2D is needed | Standard tier |
| No device-side buffering | Store-and-forward |
| Owner-level shared access policies for apps | Least-privilege policies or Entra RBAC |

## Check yourself

1. What messaging patterns does IoT Hub support?
2. What are device twins?
3. What does DPS do, and which attestation methods exist?
4. Name the DPS allocation policies.
5. When would you choose IoT Operations instead?
6. How do you route telemetry?

<details>
<summary>Answers</summary>

1. Device-to-cloud telemetry, cloud-to-device messages, direct methods, device twin updates, file upload.
2. JSON documents with desired and reported properties, tags and metadata for device state and configuration.
3. Zero-touch provisioning assigning devices to hubs; X.509, TPM and symmetric keys.
4. Evenly weighted, lowest latency, static, custom.
5. For industrial edge data planes on Arc-enabled Kubernetes with local MQTT and connectors.
6. Message routing to endpoints (Event Hubs, Service Bus, Storage, Cosmos DB) with conditions and enrichments.

</details>

## References

- [What is Azure IoT Hub?](https://learn.microsoft.com/azure/iot-hub/iot-concepts-and-iot-hub)
- [What is Device Provisioning Service?](https://learn.microsoft.com/azure/iot-dps/about-iot-dps)
- [IoT Hub message routing](https://learn.microsoft.com/azure/iot-hub/iot-hub-devguide-messages-d2c)
- [Device twins](https://learn.microsoft.com/azure/iot-hub/iot-hub-devguide-device-twins)
