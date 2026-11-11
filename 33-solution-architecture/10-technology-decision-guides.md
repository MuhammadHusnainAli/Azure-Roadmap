# Technology Decision Guides

## Why it matters

Azure offers several services for almost every need: at least six ways to run code, a dozen data stores, four messaging services, four load balancers and many ways to build AI. Architects are judged on choosing well and explaining why. This file consolidates the decision trees used throughout the roadmap into one reference, based on the Azure Architecture Center's technology decision guides, so you can answer "which service should we use?" quickly and defensibly, then dive into the detailed topic files.

## Key concepts

### Compute

| Need | Choose |
|---|---|
| Web app/API with minimal ops | **App Service** |
| Containerised microservices, event-driven scale, scale to zero | **Container Apps** |
| Full Kubernetes control, complex platforms | **AKS** |
| Event-driven functions, short tasks | **Azure Functions** |
| Lift-and-shift, full OS control, special software | **Virtual Machines / VMSS** |
| Large-scale parallel batch/HPC | **Azure Batch**, HPC VMs |
| Static front ends | **Static Web Apps** |
| Workflows/integration with connectors | **Logic Apps** |

Key questions: control vs managed, containers or code, scale pattern, team skills ([Topics 06, 07, 10](../10-application-platform/README.md)).

### Data stores

| Need | Choose |
|---|---|
| Relational, SQL Server compatibility | **Azure SQL Database / SQL MI** |
| Open-source relational | **Azure Database for PostgreSQL / MySQL** |
| Globally distributed NoSQL, low latency, flexible schema | **Cosmos DB** |
| MongoDB compatibility | **Azure DocumentDB** or Cosmos DB for MongoDB |
| Key-value cache, sessions | **Azure Managed Redis** |
| Objects, files, data lake | **Blob Storage / ADLS Gen2**, **Azure Files** |
| Analytics warehouse/lakehouse | **Microsoft Fabric**, **Azure Databricks** |
| Time series and log analytics | **Azure Data Explorer / Fabric Eventhouse** |
| Search and vector retrieval | **Azure AI Search**; vector support in Cosmos DB, PostgreSQL, SQL |
| Graph | Cosmos DB for Apache Gremlin, PostgreSQL with extensions |

([Topics 08, 09, 12](../09-databases/01-database-decision-guide.md))

### Messaging

| Need | Choose |
|---|---|
| Reliable enterprise messaging, queues/topics, transactions, sessions | **Service Bus** |
| Reactive event distribution (discrete events, pub/sub, MQTT) | **Event Grid** |
| High-throughput event streaming, telemetry, Kafka-compatible | **Event Hubs** |
| Simple, cheap queues | **Storage queues** |

([Topic 11 file 01](../11-integration-messaging-and-apis/01-messaging-decision-guide.md))

### Load balancing and traffic

| Scope / layer | Choose |
|---|---|
| Global HTTP(S) with WAF, caching, acceleration | **Azure Front Door** |
| Global DNS-based for any protocol | **Traffic Manager** |
| Regional HTTP(S) with WAF, path routing, private backends | **Application Gateway** |
| Regional/zonal layer 4 | **Azure Load Balancer** (cross-region LB for global L4) |
| API management, security, AI gateway | **API Management** |

([Topic 05 file 07](../05-networking/07-load-balancing-decision-guide.md))

### AI

| Need | Choose |
|---|---|
| Productivity for Microsoft 365 users | **Microsoft 365 Copilot** / Copilot extensibility |
| Low-code agents and workflows | **Copilot Studio** |
| Custom AI apps and agents, model choice, evaluation | **Microsoft Foundry** (Agent Service, Models, Foundry IQ) |
| Prebuilt vision, speech, language, document extraction | **Foundry Tools** (Azure AI services) |
| Knowledge retrieval / RAG | **Azure AI Search / Foundry IQ** |
| Custom ML models with MLOps | **Azure Machine Learning** |
| On-device / edge inference | **Foundry Local**, Arc-enabled AI extensions |
| Agent orchestration in code | **Microsoft Agent Framework** |

([Topic 24 file 05](../24-copilot-and-low-code-agents/05-build-vs-buy-decision-guide.md), [Topic 20 file 11](../20-foundry-tools/11-foundry-tools-vs-llm-decision-guide.md), [Topic 34](../34-ai-solution-architecture/README.md))

### Identity and secrets

| Need | Choose |
|---|---|
| Workforce identity | **Microsoft Entra ID** |
| Customer identity (CIAM) | **Microsoft Entra External ID** |
| Service-to-service auth | **Managed identities**; workload identity federation outside Azure |
| Agent identity | **Microsoft Entra Agent ID** |
| Secrets, keys, certificates | **Key Vault** (Managed HSM / Cloud HSM for high assurance) |

### Decision process

1. Start from requirements and constraints (file 03).
2. Use the decision tree to shortlist 2–3 options.
3. Evaluate against WAF pillars, skills, cost and lifecycle (Preview/retiring status).
4. Prototype if uncertain.
5. Record the choice in an ADR (file 08).

## How it works on Azure

Example decision: event ingestion for 200,000 devices sending telemetry every 10 seconds, with analytics and alerts:

```text
Volume ≈ 20,000 events/s → streaming → Event Hubs (or IoT Hub if device management needed)
Processing → Fabric Real-Time Intelligence (eventstream + Eventhouse) or Stream Analytics
Discrete alerts → Event Grid → Functions/Logic Apps → Teams/ITSM
Long-term storage → OneLake/ADLS
ADR: IoT Hub chosen because device identity, provisioning and C2D are required (not just ingestion)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Managed first** | Prefer the most managed service that meets requirements. |
| **Fewer services** | Each additional service adds operational surface; reuse platforms. |
| **Skills** | The best service your team can operate beats the theoretically optimal one. |
| **Lifecycle** | Check Preview and retirement status (Topic 38). |
| **Exit and portability** | Consider open standards where switching costs matter. |

## Hands-on

1. For five scenarios (internal API, global e-commerce, IoT telemetry, document AI, data warehouse), choose services using the tables and justify each in two sentences.
2. Compare Container Apps and AKS using Microsoft's comparison page for your team.
3. Use the data store decision tree for a product catalogue and an order ledger.
4. Write an ADR for a messaging choice.
5. Check the lifecycle status of each chosen service.

## Security, cost and reliability notes

- **Security posture** varies by service (private endpoints, Entra auth, CMK support); include it in decisions.
- **Cost models** differ (provisioned vs consumption); estimate at expected scale.
- **Reliability**: check zone redundancy and multi-region capabilities of each option.

## Common mistakes

| Mistake | Fix |
|---|---|
| Choosing by familiarity only | Decision criteria |
| Event Hubs used as a work queue | Service Bus for commands |
| AKS for a few simple APIs | Container Apps/App Service |
| Custom ML for problems prebuilt AI solves | Foundry Tools first |
| Ignoring retirement notices | Lifecycle check |

## Check yourself

1. Which compute service fits containerised microservices with scale to zero and minimal ops?
2. When do you choose Service Bus vs Event Grid vs Event Hubs?
3. Which data store fits globally distributed, low-latency NoSQL?
4. Which load balancer fits global HTTP with WAF?
5. When do you choose Copilot Studio vs Foundry?
6. Describe the decision process.

<details>
<summary>Answers</summary>

1. Azure Container Apps.
2. Service Bus for reliable enterprise messaging and commands; Event Grid for reactive discrete events/pub-sub; Event Hubs for high-throughput streaming telemetry.
3. Cosmos DB.
4. Azure Front Door.
5. Copilot Studio for low-code agents by makers within Microsoft 365/Power Platform; Foundry for custom pro-code AI apps and agents with model choice and rigorous evaluation.
6. Requirements → shortlist via decision tree → evaluate with WAF, skills, cost, lifecycle → prototype if needed → ADR.

</details>

## References

- [Technology choices overview](https://learn.microsoft.com/azure/architecture/guide/technology-choices/technology-choices-overview)
- [Compute decision tree](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Data store decision tree](https://learn.microsoft.com/azure/architecture/guide/technology-choices/data-store-decision-tree)
- [Messaging options](https://learn.microsoft.com/azure/architecture/guide/technology-choices/messaging)
- [Load balancing options](https://learn.microsoft.com/azure/architecture/guide/technology-choices/load-balancing-overview)
- [Compare container options](https://learn.microsoft.com/azure/container-apps/compare-options)
