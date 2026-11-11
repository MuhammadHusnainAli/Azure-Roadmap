# Energy, Sustainability and Geospatial

## Why it matters

Energy companies manage enormous subsurface, wells, grid and asset datasets in industry formats; every organisation faces growing pressure to measure and reduce emissions (including from its cloud estate); and geospatial data (satellite imagery, maps, location telemetry) underpins agriculture, insurance, logistics, utilities and climate risk analysis. Azure offers specialised services for each: **Azure Data Manager for Energy** (a managed OSDU platform), **Microsoft Planetary Computer Pro** (geospatial data management with GeoCatalogs), **Azure Maps**, **Azure carbon optimization** and **Microsoft Sustainability Manager / sustainability data solutions in Fabric** for emissions data. This file gives architects enough depth to recognise when these services fit.

## Key concepts

### Energy: Azure Data Manager for Energy

| Aspect | Detail |
|---|---|
| **What** | Fully managed, enterprise-grade **OSDU® Data Platform** (Open Subsurface Data Universe standard) on Azure |
| **Managed by Microsoft** | Deployment, monitoring, scaling, security, updates and upgrades |
| **Security** | Encryption in transit and at rest, Entra ID authentication and authorisation |
| **Data partitions** | Multiple partitions per instance for business separation |
| **Integration** | OSDU APIs and data model; compatibility with domain applications from energy ISVs (seismic, wells, reservoir) |
| **Use cases** | Subsurface data consolidation, exploration and production workflows, AI on wells and seismic data |

### Geospatial

| Service | Purpose |
|---|---|
| **Microsoft Planetary Computer Pro** | Geospatial data management service with **GeoCatalogs**: ingest, organise (STAC-based), search and serve geospatial data at scale for solution developers, enterprise data managers and data scientists |
| **Microsoft Planetary Computer (public data)** | Open environmental datasets catalogue |
| **Azure Maps** | Location services: maps, search/geocoding, routing, weather, geolocation, indoor maps; SDKs and REST APIs ([Topic 10 file 10](../10-application-platform/10-azure-maps-and-notification-hubs.md)) |
| **Fabric and Databricks geospatial** | Spatial analytics on lakehouse data |
| **Azure OpenAI / Foundry with geospatial tools** | Agents querying spatial data via tools |

### Sustainability

| Capability | Purpose |
|---|---|
| **Azure carbon optimization** | Emissions data for your Azure resources (scope 3 for you), trends and reduction recommendations in the Azure portal |
| **Emissions Impact Dashboard** (Power BI) | Microsoft cloud emissions reporting |
| **Microsoft Sustainability Manager / sustainability data solutions in Fabric** | Collect ESG data, calculate scope 1–3 emissions, reporting for CSRD and other frameworks |
| **Sustainability APIs** | Programmatic access to cloud emissions data |
| **Well-Architected sustainability guidance** | Design principles for efficient, lower-carbon workloads |

Sustainable design practices: rightsizing and autoscale (also save cost), choosing efficient regions and SKUs, shifting batch work in time, data lifecycle policies, efficient AI model selection (smaller models, caching, batch).

## How it works on Azure

Energy: deploy an Azure Data Manager for Energy instance with data partitions; ingest well logs and seismic metadata via OSDU ingestion workflows; integrate domain apps through OSDU APIs; build analytics in Fabric/Databricks and AI agents that call OSDU search APIs as tools (with Entra-authenticated access).

Geospatial pipeline with Planetary Computer Pro:

```text
Satellite/drone imagery + vector data → ingest into GeoCatalog (STAC collections and items)
→ search by area/time/attributes → serve tiles to apps (Azure Maps / web map) → analyse in notebooks (Fabric/Databricks)
→ AI: classify land use, detect changes; results back into the catalogue
```

Cloud emissions report: open **Carbon optimization** in the Azure portal to view emissions by subscription, resource group, service and region; export data; combine with Cost Management to prioritise optimisations that cut both cost and carbon.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **OSDU platform** | Use the managed service when you've standardised on OSDU; avoid self-managing OSDU on Kubernetes unless necessary. |
| **Geospatial store** | Planetary Computer Pro for managed STAC catalogues of imagery; lakehouse spatial for analytics-heavy vector data. |
| **Emissions scope** | Cloud emissions are one part of scope 3; use sustainability solutions for enterprise-wide accounting. |
| **Sustainability vs cost** | Usually aligned (efficiency); occasionally region choices trade carbon intensity against latency or residency. |
| **AI efficiency** | Prefer smaller models and caching where quality allows. |

## Hands-on

1. Review Azure carbon optimization data for a subscription and identify the top emitting services.
2. Design an energy data architecture with Azure Data Manager for Energy, Fabric and an AI assistant over OSDU data.
3. Create a Planetary Computer Pro GeoCatalog in a test subscription and ingest sample STAC data.
4. Build a simple map app with Azure Maps showing assets and routes.
5. Write a sustainability section for an architecture decision record.

## Security, cost and reliability notes

- **Sensitive data**: subsurface data and asset locations can be commercially or nationally sensitive; restrict access and network exposure.
- **Cost**: specialised services have instance-based pricing; plan environments carefully.
- **Data quality**: emissions and geospatial outputs depend on input data quality; document methodologies.

## Common mistakes

| Mistake | Fix |
|---|---|
| Self-hosting OSDU without need | Managed Azure Data Manager for Energy |
| Treating cloud emissions as total footprint | Enterprise sustainability accounting |
| Storing imagery without a catalogue | STAC GeoCatalogs |
| Ignoring sustainability in design | Well-Architected sustainability guidance |
| Using large models for every task | Right-size models |

## Check yourself

1. What is Azure Data Manager for Energy?
2. What does Microsoft Planetary Computer Pro provide?
3. Which services help measure emissions?
4. How do cost and sustainability optimisation relate?
5. What is STAC, and why use it?
6. Name three sustainable design practices.

<details>
<summary>Answers</summary>

1. A fully managed OSDU Data Platform for energy data with Microsoft-managed operations, Entra security and data partitions.
2. Geospatial data management with GeoCatalogs to ingest, organise, search and serve geospatial data at scale.
3. Azure carbon optimization, Emissions Impact Dashboard, sustainability APIs, Microsoft Sustainability Manager/sustainability data solutions in Fabric.
4. Mostly aligned: efficiency reduces both; occasionally region choices trade carbon against latency or residency.
5. SpatioTemporal Asset Catalog, a standard for describing geospatial assets so they can be searched and used consistently.
6. Any three: rightsizing/autoscale, efficient regions and SKUs, time-shifting batch work, data lifecycle policies, efficient AI model use.

</details>

## References

- [What is Azure Data Manager for Energy?](https://learn.microsoft.com/azure/energy-data-services/overview-microsoft-energy-data-services)
- [What is Microsoft Planetary Computer Pro?](https://learn.microsoft.com/azure/planetary-computer/microsoft-planetary-computer-pro-overview)
- [Azure carbon optimization](https://learn.microsoft.com/azure/carbon-optimization/overview)
- [Sustainability data solutions in Fabric](https://learn.microsoft.com/industry/sustainability/sustainability-data-solutions-overview)
- [Well-Architected sustainability](https://learn.microsoft.com/azure/well-architected/sustainability/)
- [Azure Maps](https://learn.microsoft.com/azure/azure-maps/about-azure-maps)
