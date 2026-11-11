# Azure Digital Twins

## Why it matters

Telemetry on its own says "sensor 4711 reads 82 °C". Operations need context: which room, floor, building, production line and process that sensor belongs to, what equipment is upstream, which tenant occupies the space, and what that temperature means for energy cost or product quality. **Azure Digital Twins** models entire environments (buildings, factories, energy grids, campuses, supply chains) as a **twin graph** of entities and relationships defined in a modelling language, keeps it current with live data, and lets applications query and react to the state of the environment. Microsoft Fabric also offers **digital twin builder** (Preview) in Real-Time Intelligence, so architects should know when each fits.

## Key concepts

### Building blocks

| Concept | Meaning |
|---|---|
| **DTDL models** | Digital Twins Definition Language (JSON-LD) defining twin types: properties, telemetry, components, relationships |
| **Twins** | Instances of models (for example `Room-101`, `Pump-P12`) with property values |
| **Relationships** | Directed links between twins (`contains`, `feeds`, `isLocatedIn`) forming a graph |
| **Twin graph** | The live model of the environment |
| **Query language** | SQL-like queries over twins and relationships (for example all rooms on floor 3 above 26 °C) |
| **Event routes** | Twin changes and lifecycle events to Event Hubs, Event Grid or Service Bus |
| **Data history** | Historise twin changes to Azure Data Explorer for time-series analysis |
| **Ontologies** | Industry ontologies (RealEstateCore for buildings, energy grid, smart cities) to start from |

### Distinctions

| Term | What it is |
|---|---|
| **IoT Hub device twin** | Per-device state/configuration document for device management |
| **Azure Digital Twins** | Environment-level graph model with relationships and queries (PaaS) |
| **Fabric digital twin builder** (Preview) | Modelling and contextualisation of operational data inside Fabric Real-Time Intelligence, integrated with OneLake and analytics |

### Typical integration

```text
Devices → IoT Hub / IoT Operations → (Azure Function or data flow updates twin properties) → Azure Digital Twins
Azure Digital Twins → event routes → downstream processing, alerts; data history → Azure Data Explorer / Fabric
Apps and 3D visualisations query the twin graph; AI agents can use twin queries as tools for contextual answers
```

## How it works on Azure

DTDL model (excerpt):

```json
{
  "@id": "dtmi:contoso:Room;1",
  "@type": "Interface",
  "@context": "dtmi:dtdl:context;3",
  "displayName": "Room",
  "contents": [
    { "@type": "Property", "name": "temperature", "schema": "double" },
    { "@type": "Property", "name": "occupied", "schema": "boolean" },
    { "@type": "Relationship", "name": "hasSensor", "target": "dtmi:contoso:Sensor;1" }
  ]
}
```

Create instance, upload models and twins (CLI with the `azure-iot` extension):

```bash
az dt create -g rg-twins -n adt-campus -l westeurope --assign-identity
az dt model create -n adt-campus --from-directory ./models
az dt twin create -n adt-campus --dtmi "dtmi:contoso:Room;1" --twin-id Room-101 --properties '{"temperature":21.5,"occupied":false}'
az dt twin relationship create -n adt-campus --relationship-id r1 --relationship hasSensor --twin-id Room-101 --target Sensor-4711
```

Query hot rooms:

```sql
SELECT room FROM DIGITALTWINS room WHERE IS_OF_MODEL(room, 'dtmi:contoso:Room;1') AND room.temperature > 26
```

Grant applications **Azure Digital Twins Data Reader/Owner** roles with Entra identities; ingestion functions use managed identity.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **ADT vs Fabric digital twin builder** | ADT for application-centric, API-driven twin graphs with event routing; Fabric option (Preview) for analytics-centric contextualisation within Fabric. |
| **Model design** | Start from industry ontologies; version models (`;1`, `;2`); keep graphs focused on decisions you need to support. |
| **Ingestion** | Update twins with significant changes, not every telemetry sample; keep raw telemetry in time-series stores. |
| **History** | Data history to ADX/Fabric for trends and ML. |
| **AI** | Expose twin queries via tools/MCP so agents can answer contextual operational questions. |

## Hands-on

1. Write DTDL models for building, floor, room and sensor; upload them.
2. Create a small twin graph and query it.
3. Build an Azure Function that updates twin properties from IoT Hub telemetry.
4. Configure an event route and data history to Azure Data Explorer.
5. Compare the same scenario in Fabric digital twin builder.

## Security, cost and reliability notes

- **RBAC** with Entra; private endpoints for the instance.
- **Cost**: operations, queries and messages; avoid excessive update frequency.
- **Reliability**: regional service; plan backup of models and graph definitions as code.
- **Data sensitivity**: occupancy and location data can be personal; govern it.

## Common mistakes

| Mistake | Fix |
|---|---|
| Updating twins with every telemetry message | Update on meaningful change |
| Inventing ontologies from scratch | Start from industry ontologies |
| Confusing device twins with Digital Twins | Understand the distinction |
| Graph defined manually in the portal | Models and twins as code |
| No history store | Data history to ADX/Fabric |

## Check yourself

1. What is a twin graph?
2. What does DTDL define?
3. How do IoT Hub device twins differ from Azure Digital Twins?
4. How do twins get updated with live data?
5. What are event routes and data history?
6. When might Fabric digital twin builder be preferable?

<details>
<summary>Answers</summary>

1. A live graph of twins (entities) and relationships modelling an environment.
2. Twin types: properties, telemetry, components and relationships.
3. Device twins hold per-device state for device management; Azure Digital Twins models environments with relationships and queries.
4. Through functions or data flows that process telemetry and update twin properties.
5. Event routes send twin change events to Event Hubs/Event Grid/Service Bus; data history stores twin changes in Azure Data Explorer.
6. For analytics-centric contextualisation within Fabric and OneLake (Preview).

</details>

## References

- [What is Azure Digital Twins?](https://learn.microsoft.com/azure/digital-twins/overview)
- [DTDL models](https://learn.microsoft.com/azure/digital-twins/concepts-models)
- [Digital twin builder in Fabric (Preview)](https://learn.microsoft.com/fabric/real-time-intelligence/digital-twin-builder/overview)
- [Data history](https://learn.microsoft.com/azure/digital-twins/concepts-data-history)
