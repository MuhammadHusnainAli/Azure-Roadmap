# Azure IoT Operations and IoT Edge

## Why it matters

Industrial sites run a mix of PLCs, OPC UA servers, historians, cameras and proprietary protocols, and the operational technology (OT) world has traditionally been isolated from IT and cloud. Manufacturers want to normalise that data at the edge, act on it locally, and send curated streams to the cloud for analytics and AI without sending every raw sample. **Azure IoT Operations** is a unified edge data plane running on **Arc-enabled Kubernetes**, with an industrial-grade **MQTT broker**, **connectors** (including OPC UA), **data flows** and asset management, integrated with Event Hubs, Event Grid's MQTT broker and Microsoft Fabric. **Azure IoT Edge** remains the device-focused container runtime for individual edge devices connected to IoT Hub.

## Key concepts

### Azure IoT Operations

| Component | Purpose |
|---|---|
| **MQTT broker** | Edge-native, industrial-grade, highly available broker powering event-driven architectures |
| **Connectors** | Connector for OPC UA, media/ONVIF and other connectors to bring asset data into the broker |
| **Data flows** | Transform, filter, enrich, normalise and route data between the broker and cloud endpoints (Event Hubs, Event Grid MQTT, Fabric, Data Lake Storage) |
| **Assets and devices** | Model and manage industrial assets (Azure Device Registry); discover OPC UA assets |
| **Operations experience** | Web UI for OT users to configure assets and data flows |
| **Security** | Secrets management, certificate management, secure settings; Arc-managed |
| **Offline** | Operates offline for up to **72 hours** with possible degradation; resumes on reconnection |
| **Platform** | Kubernetes-native on Arc-enabled clusters (for example AKS on Azure Local) |

### Azure IoT Edge

| Aspect | Detail |
|---|---|
| **What** | Device runtime deploying containerised Linux modules managed from IoT Hub |
| **Components** | IoT Edge runtime (security daemon, edge agent, edge hub), modules, cloud interface (IoT Hub deployments) |
| **Versions** | **IoT Edge 1.6 LTS** is the supported release; **1.5 LTS support ends 10 November 2026**; 1.4 reached end of life in November 2024 |
| **Use** | Single gateways and devices running analytics, protocol translation or ML inference with offline capability |

### Choosing

| Scenario | Fit |
|---|---|
| Factory-wide OT/IT data plane with many assets, OPC UA, MQTT, Kubernetes infrastructure | **IoT Operations** |
| Individual devices or gateways needing containerised logic, managed per device from IoT Hub | **IoT Edge** |
| Cloud-connected device fleets without edge processing | **IoT Hub** (file 01) |
| MQTT pub/sub in the cloud | **Event Grid MQTT broker** |

## How it works on Azure

Deploy IoT Operations on an Arc-enabled cluster (outline; follow the current quickstart):

```bash
az connectedk8s connect -g rg-factory01 -n aks-factory01-arc -l westeurope
az iot ops schema registry create -g rg-factory01 -n sr-factory01 --registry-namespace srfactory01 \
  --sa-resource-id <storage-account-id>
az iot ops init -g rg-factory01 --cluster aks-factory01-arc
az iot ops create -g rg-factory01 --cluster aks-factory01-arc --name aio-factory01 \
  --sr-resource-id <schema-registry-id>
```

Data flow concept: OPC UA connector reads tags from a CNC machine → MQTT topic `factory01/line2/cnc7/telemetry` → data flow filters and converts units, adds asset metadata → sends to Fabric eventstream (via Event Hubs) for Real-Time Intelligence dashboards and to a local topic for an edge anomaly model.

IoT Edge deployment manifest (excerpt) for a gateway module:

```json
{
  "modulesContent": {
    "$edgeAgent": { "properties.desired": { "modules": {
      "opcPublisher": { "type": "docker", "status": "running", "restartPolicy": "always",
        "settings": { "image": "mcr.microsoft.com/iotedge/opc-publisher:2.9" } } } } }
  }
}
```

Apply with `az iot edge set-modules --hub-name <hub> --device-id gw-01 --content deployment.json`.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Edge platform** | IoT Operations on Arc Kubernetes for site-wide platforms; IoT Edge for discrete devices. |
| **Data reduction** | Normalise and aggregate at the edge; send events and summaries to the cloud. |
| **Cloud destination** | Fabric Real-Time Intelligence for analytics and dashboards; Event Hubs for custom processing; Digital Twins for contextual models (file 03). |
| **OT security** | Network segmentation (Purdue model layers), certificates, least privilege; Defender for IoT. |
| **Lifecycle** | Keep IoT Edge on 1.6 LTS; plan migrations before support ends. |
| **Offline** | Design for at least the documented offline window; buffer locally. |

## Hands-on

1. Deploy IoT Operations on a test Arc-enabled cluster with the quickstart and simulated OPC PLC.
2. Create an asset and data flow sending normalised data to Event Hubs and on to Fabric.
3. Build a real-time dashboard in Fabric.
4. Deploy an IoT Edge 1.6 device with a module and update it via deployment manifest.
5. Check any devices on IoT Edge 1.5 and plan upgrades before 10 November 2026.

## Security, cost and reliability notes

- **OT/IT boundary**: never expose OT networks directly; use layered networks and brokers.
- **Certificates and secrets** managed by the platform; rotate.
- **Cost**: Arc-enabled Kubernetes infrastructure, IoT Operations pricing, cloud ingestion.
- **Reliability**: HA broker configuration; site hardware redundancy.

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending raw high-frequency data to the cloud | Edge data flows |
| Using IoT Edge for site-wide platforms | IoT Operations on Kubernetes |
| Running unsupported IoT Edge versions | Upgrade to 1.6 LTS |
| Flat OT networks | Segmentation |
| No offline design | Buffering and local logic |

## Check yourself

1. What is Azure IoT Operations, and where does it run?
2. Name its main components.
3. How long can IoT Operations run offline?
4. What is IoT Edge, and which version is supported?
5. When do you choose IoT Operations vs IoT Edge vs IoT Hub alone?
6. Where should IoT Operations data go in the cloud?

<details>
<summary>Answers</summary>

1. A unified edge data plane of modular data services on Arc-enabled Kubernetes.
2. MQTT broker, connectors (OPC UA and others), data flows, asset/device management, operations experience, security services.
3. Up to 72 hours, possibly with degradation.
4. A device runtime for containerised modules managed from IoT Hub; IoT Edge 1.6 LTS (1.5 support ends 10 November 2026).
5. IoT Operations for site-wide industrial data planes; IoT Edge for individual devices/gateways; IoT Hub alone for cloud-connected fleets without edge processing.
6. Event Hubs, Event Grid MQTT broker, Microsoft Fabric or Data Lake Storage depending on use.

</details>

## References

- [What is Azure IoT Operations?](https://learn.microsoft.com/azure/iot-operations/overview-iot-operations)
- [IoT Operations MQTT broker](https://learn.microsoft.com/azure/iot-operations/manage-mqtt-broker/overview-broker)
- [IoT Operations data flows](https://learn.microsoft.com/azure/iot-operations/connect-to-cloud/overview-dataflow)
- [What is Azure IoT Edge?](https://learn.microsoft.com/azure/iot-edge/about-iot-edge)
- [Event Grid MQTT broker](https://learn.microsoft.com/azure/event-grid/mqtt-overview)
