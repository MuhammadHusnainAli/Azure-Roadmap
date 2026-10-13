# Azure Service Map

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure has hundreds of services and several that overlap. Before learning any of them in depth you need a map: what categories exist, what the main service in each is, and roughly when you would pick it. This file is that map. Come back to it whenever a later topic mentions a service you don't recognise.

Each section points to the topic that covers it properly.

## How to read the map

Azure services group into layers. A typical solution takes something from most of them.

```text
┌──────────────────────────────────────────────────────────────────────┐
│  Experiences     Microsoft 365 Copilot · Copilot Studio · your apps   │
├──────────────────────────────────────────────────────────────────────┤
│  AI              Foundry · Models · Agent Service · AI Search · Tools │
├──────────────────────────────────────────────────────────────────────┤
│  Data            Fabric · Databricks · SQL · Cosmos DB · PostgreSQL   │
├──────────────────────────────────────────────────────────────────────┤
│  App platform    App Service · Functions · Container Apps · AKS       │
│  Integration     API Management · Service Bus · Event Grid · Logic Apps│
├──────────────────────────────────────────────────────────────────────┤
│  Infrastructure  Compute · Storage · Networking                       │
├──────────────────────────────────────────────────────────────────────┤
│  Foundation      Entra ID · Policy · Monitor · Defender · Key Vault   │
│  (cross-cutting) Cost Management · DevOps · Backup                    │
└──────────────────────────────────────────────────────────────────────┘
```

## Identity → topic 02

| Service | What it is | Use when |
|---|---|---|
| **Microsoft Entra ID** | Cloud identity and access management | Always. Every Azure solution uses it |
| **Managed identities** | Automatic identities for Azure resources | A service needs to call another service without secrets |
| **Entra External ID** | Identity for customers and partners | Building apps for people outside your organisation |
| **Entra Domain Services** | Managed AD DS (Kerberos, LDAP, NTLM) | Legacy apps need a domain and you don't want domain controllers |
| **Entra Agent ID** | Identities for AI agents | Governing what agents can access |

## Governance and management → topics 03, 26, 29

| Service | What it is |
|---|---|
| **Management groups, Azure Policy** | Organise subscriptions and enforce rules |
| **Azure Resource Graph** | Query the whole estate with KQL |
| **Azure Monitor** (Log Analytics, Application Insights) | Metrics, logs, traces, alerts |
| **Azure Advisor** | Recommendations for cost, security, reliability, performance |
| **Azure Service Health** | Incidents and maintenance affecting your resources |
| **Microsoft Cost Management** | Cost analysis, budgets, exports |
| **Azure Update Manager** | OS patching for Azure and Arc-enabled machines |
| **Azure Automation** | Runbooks and scheduled operational tasks |
| **Azure Lighthouse** | Manage customer tenants from your own |
| **Azure Copilot, Azure SRE Agent** | AI-assisted operations |

## Networking → topic 05

| Need | Service |
|---|---|
| Private network in Azure | **Virtual Network**, subnets, NSGs |
| Connect VNets | **VNet peering**, **Virtual WAN**, **Virtual Network Manager** |
| Connect on-premises | **VPN Gateway** (internet), **ExpressRoute** (private circuit) |
| Private access to PaaS | **Private Link / private endpoints** |
| Name resolution | **Azure DNS**, **DNS Private Resolver** |
| Outbound internet | **NAT Gateway** |
| Central inspection and filtering | **Azure Firewall** |
| Secure admin access to VMs | **Azure Bastion** |
| DDoS mitigation | **Azure DDoS Protection** |

**Load balancing: the four that get confused**

| Service | Layer | Scope | Use for |
|---|---|---|---|
| **Load Balancer** | 4 (TCP/UDP) | Regional (global tier exists) | Non-HTTP traffic, internal tiers |
| **Application Gateway** | 7 (HTTP) | Regional | Web apps in one region, with WAF, path routing, TLS termination |
| **Front Door** | 7 (HTTP) | Global | Multi-region web apps, edge acceleration, WAF, CDN |
| **Traffic Manager** | DNS | Global | DNS-based routing for any protocol |

Rule of thumb: global and HTTP is Front Door; regional and HTTP is Application Gateway; non-HTTP is Load Balancer; DNS-level failover is Traffic Manager.

## Compute → topics 06, 07, 10

| Service | Model | Use when |
|---|---|---|
| **Virtual Machines** | IaaS | You need OS control, legacy software, or specific hardware such as GPUs |
| **Virtual Machine Scale Sets** | IaaS | Many identical VMs with autoscale |
| **Azure Batch** | Managed job scheduling on VMs | Large parallel and HPC jobs |
| **App Service** | PaaS | Web apps and APIs with minimal operations |
| **Azure Functions** | Serverless | Event-driven code, short tasks, glue logic, tool back ends for agents |
| **Azure Container Apps** | Serverless containers | Microservices, APIs, jobs and AI apps in containers without managing Kubernetes |
| **Azure Kubernetes Service (AKS)** | Managed Kubernetes | You need the Kubernetes API, ecosystem or fine control |
| **Azure Container Instances** | Single containers on demand | Simple, isolated, short-lived containers |
| **Static Web Apps** | Managed static hosting | Front ends with optional serverless API |
| **Azure Virtual Desktop** | Desktop virtualisation | Remote desktops and apps |
| **Azure VMware Solution** | Managed VMware | Moving VMware estates without re-platforming |

**Quick compute decision**

```text
Need full OS control or special hardware?           → Virtual Machines
Event-driven, short-lived functions?                → Azure Functions
Standard web app or API, code deployment?           → App Service
Containers, no Kubernetes management wanted?        → Container Apps
Need Kubernetes API, operators, custom networking?  → AKS
Static front end?                                   → Static Web Apps
```

## Storage → topic 08

| Service | Data shape | Use for |
|---|---|---|
| **Blob Storage** | Objects | Documents, images, backups, RAG source files, model artefacts |
| **Data Lake Storage** | Objects with hierarchical namespace | Analytics data lakes |
| **Azure Files** | SMB / NFS file shares | Lift-and-shift file shares, shared app storage |
| **Managed Disks** | Block | VM disks |
| **Queue Storage** | Simple messages | Basic decoupling |
| **Table Storage** | Key-value | Simple, cheap structured data |
| **Azure NetApp Files** | Enterprise file | High-performance file workloads, SAP, HPC |
| **Elastic SAN** | Block at scale | Consolidated block storage for many workloads |
| **Managed Lustre** | Parallel file system | HPC and large model training |

Redundancy options appear everywhere: **LRS** (three copies in one data centre), **ZRS** (three zones), **GRS / GZRS** (plus a second region).

## Databases → topic 09

| Service | Type | Use when |
|---|---|---|
| **Azure SQL Database** | Relational (SQL Server engine) | New relational apps on the Microsoft stack |
| **Azure SQL Managed Instance** | Relational, near-full SQL Server compatibility | Migrating SQL Server with minimal change |
| **Azure Database for PostgreSQL** | Relational, open source | PostgreSQL apps; strong vector support with pgvector |
| **Azure Database for MySQL** | Relational, open source | MySQL apps |
| **Azure Cosmos DB** | NoSQL, globally distributed | Low-latency, high-scale, multi-region; chat history and agent state |
| **Azure DocumentDB** | MongoDB-compatible | MongoDB workloads |
| **Azure Managed Redis** | In-memory | Caching, sessions, semantic cache |
| **Azure Managed Instance for Apache Cassandra** | Wide-column | Cassandra workloads |

Most of these now support vector search. Whether to use them or Azure AI Search for retrieval is covered in topics 09 and 17.

## Integration and messaging → topic 11

| Service | Pattern | Use for |
|---|---|---|
| **API Management** | API gateway | Publish, secure, throttle and observe APIs; AI gateway in front of models; governing MCP servers |
| **Service Bus** | Enterprise message broker | Reliable commands and transactions between services |
| **Event Grid** | Event routing (pub/sub) | Reacting to things that happened; MQTT |
| **Event Hubs** | Event streaming | High-volume telemetry and streams; Kafka-compatible |
| **Logic Apps** | Workflow with connectors | Integrating SaaS and enterprise systems with little code |
| **Data Factory** | Data pipelines | Moving and transforming data on a schedule |

**Messaging decision in one line each**

- *Something happened, tell whoever cares:* Event Grid.
- *Millions of events per second to analyse:* Event Hubs.
- *Do this, exactly once, in order, and don't lose it:* Service Bus.
- *Cheap, simple queue:* Storage Queues.

## Data and analytics → topic 12

| Service | What it is |
|---|---|
| **Microsoft Fabric** | Unified SaaS analytics platform: OneLake, lakehouse, warehouse, real-time intelligence, Data Factory, Power BI |
| **Azure Databricks** | Spark-based lakehouse platform for data engineering and ML |
| **Power BI** | Business intelligence and reporting |
| **Azure Data Explorer** | Fast analytics on logs and telemetry with KQL |
| **Azure Stream Analytics** | SQL-based stream processing |
| **Azure Synapse Analytics** | Earlier-generation integrated analytics; new work typically goes to Fabric |
| **Microsoft Purview** | Data governance, catalogue, compliance |

## AI → topics 13 to 24

This is the centre of the roadmap. The names changed a lot in 2025; former names are shown once.

**Platform**

| Service | What it is |
|---|---|
| **Microsoft Foundry** (formerly Azure AI Foundry, Azure AI Studio) | The platform for building AI apps and agents: models, agents, tools, evaluation, governance |
| **Foundry Models** | The model catalogue and inference service: OpenAI, Anthropic, Microsoft, Meta, Mistral, xAI and others |
| **Azure OpenAI in Foundry Models** (formerly Azure OpenAI Service) | OpenAI models hosted by Azure |
| **Foundry Agent Service** | Managed runtime for agents: tools, memory, orchestration, publishing |
| **Foundry IQ** | Knowledge layer for grounding agents across enterprise data |
| **Foundry Control Plane** | Fleet-wide visibility, guardrails and governance for agents and models |
| **Foundry Local** | Run models on device |
| **Azure AI Search** | Search and retrieval engine: keyword, vector, hybrid, agentic retrieval |
| **Azure Machine Learning** | Train, deploy and manage custom ML models |

**Foundry Tools** (formerly Azure AI services, Cognitive Services)

| Service | Does |
|---|---|
| **Azure Speech** | Speech to text, text to speech, translation, voice agents |
| **Azure Vision** | Image analysis, OCR, face |
| **Azure Language** | Entity extraction, PII detection, summarisation, classification |
| **Azure Translator** | Text and document translation |
| **Azure Document Intelligence** | Structured extraction from forms and documents |
| **Azure Content Understanding** | Multimodal extraction from documents, images, audio, video |
| **Azure AI Content Safety** | Harm detection, prompt shields, groundedness checks |
| **Azure AI Video Indexer** | Insights from video and audio |

**Developer frameworks and protocols**

| Item | What it is |
|---|---|
| **Microsoft Agent Framework** | Open-source SDK for agents and multi-agent workflows; successor to Semantic Kernel and AutoGen |
| **Model Context Protocol (MCP)** | Open standard for connecting agents to tools and data |
| **Agent-to-Agent (A2A)** | Open standard for agents to call other agents |

**Copilots and low-code**

| Service | What it is |
|---|---|
| **Microsoft 365 Copilot** | AI assistant across Microsoft 365, grounded in your organisation's data |
| **Copilot Studio** | Low-code builder for agents |
| **Agent 365** | Control plane for managing agents across the organisation |
| **GitHub Copilot** | AI coding assistant and coding agent |

**Which AI building route?**

| If you want to… | Start with |
|---|---|
| Give employees an assistant over Microsoft 365 data, no build | Microsoft 365 Copilot |
| Build a business agent with little code, on Microsoft 365 and Power Platform connectors | Copilot Studio |
| Build a custom AI app or agent with full control of models, retrieval, tools and hosting | Microsoft Foundry |
| Train your own predictive ML model | Azure Machine Learning or Fabric |
| Extract fields from documents or transcribe audio | The matching Foundry Tool |

## Security → topics 23, 25

| Service | What it is |
|---|---|
| **Microsoft Defender for Cloud** | Posture management and workload protection, including AI workloads |
| **Microsoft Sentinel** | Cloud-native SIEM and SOAR |
| **Azure Key Vault** | Secrets, keys and certificates |
| **Azure Firewall, WAF, DDoS Protection** | Network protection |
| **Microsoft Purview** | Data security, classification, DLP, compliance |
| **Microsoft Security Copilot** | AI assistant and agents for security teams |

## DevOps and developer platform → topics 04, 27

| Service | What it is |
|---|---|
| **GitHub** (Actions, Advanced Security, Copilot) | Source, CI/CD, security scanning, AI assistance |
| **Azure DevOps** | Boards, Repos, Pipelines, Artifacts, Test Plans |
| **Bicep, Terraform, Azure Developer CLI** | Infrastructure as code and app deployment |
| **Azure Container Registry** | Private container image registry |
| **Azure App Configuration** | Central configuration and feature flags |
| **Microsoft Dev Box, Deployment Environments** | Self-service developer workstations and environments |
| **Azure App Testing, Chaos Studio** | Load, end-to-end and resilience testing |

## Resilience and migration → topics 28, 30

| Service | What it is |
|---|---|
| **Azure Backup** | Backup for VMs, databases, files, blobs |
| **Azure Site Recovery** | Replication and failover for disaster recovery |
| **Azure Migrate** | Discovery, assessment and migration of servers, databases and apps |
| **Database Migration Service** | Database migrations |
| **Data Box, Storage Mover** | Bulk and ongoing data transfer |

## Hybrid, edge and specialised → topics 31, 32

| Service | What it is |
|---|---|
| **Azure Arc** | Manage servers, Kubernetes and data services anywhere from Azure |
| **Azure Local** (formerly Azure Stack HCI) | Azure services on your own hardware |
| **Azure IoT Hub, IoT Operations** | Connect and manage devices and industrial assets |
| **Azure Digital Twins** | Model physical environments |
| **Azure Communication Services** | Voice, video, chat, SMS, email APIs |
| **Azure Maps** | Geospatial APIs |
| **Azure Health Data Services** | FHIR and DICOM health data platform |
| **Azure Quantum** | Quantum hardware and tooling |

## Reference stack: a typical enterprise AI application

Seeing the pieces assembled helps the map stick.

| Concern | Service |
|---|---|
| User sign-in | Microsoft Entra ID |
| Global entry and WAF | Azure Front Door |
| Front end and API | Azure Container Apps or App Service |
| API and model gateway | Azure API Management |
| Orchestration and agents | Foundry Agent Service, Microsoft Agent Framework |
| Models | Foundry Models |
| Retrieval | Azure AI Search / Foundry IQ |
| Source documents | Blob Storage |
| Document parsing | Azure Document Intelligence or Content Understanding |
| Conversation and agent state | Azure Cosmos DB |
| Secrets and keys | Azure Key Vault, managed identities |
| Safety | Azure AI Content Safety, Foundry guardrails |
| Private connectivity | Virtual Network, private endpoints |
| Observability | Azure Monitor, Application Insights, Foundry tracing and evaluations |
| Security posture | Defender for Cloud, Microsoft Purview |
| Delivery | GitHub Actions, Bicep or Terraform |

Topic 34 turns this into reference architectures.

## Design decisions and trade-offs

- **Prefer one well-understood service to three clever ones.** Every service added is another thing to secure, monitor, pay for and explain.
- **Overlap is real.** Container Apps vs AKS, Service Bus vs Event Grid, Fabric vs Databricks, Foundry vs Copilot Studio. Use the decision guides in the relevant topics rather than habit.
- **Check the region.** Not every service or SKU is in every region.
- **Check the lifecycle.** Some names you will see in older material are retired or retiring. Topic 38 lists them.

## Hands-on

1. Open **All services** in the portal and browse each category. For each, favourite the one service you are most likely to use.
2. For three services you have never used, open the create wizard, read every tab, and cancel on the review page. Note the pricing tiers offered.
3. Take a solution you know (from work or a tutorial) and fill in the reference stack table above with what it actually uses. Note any concern that has no service against it.
4. In the Foundry portal, open the model catalogue and filter by provider. Count how many providers are available to you.

## Common mistakes

- Reaching for AKS by default when Container Apps or App Service would do.
- Using Event Hubs as a message queue, or Service Bus for telemetry streams.
- Building a custom solution in Foundry when Microsoft 365 Copilot or Copilot Studio already meets the need, or the reverse.
- Using a large language model for a task a Foundry Tool does more cheaply and accurately, such as OCR or translation.
- Learning service names from old material and proposing something retired.
- Confusing Azure Monitor (operations telemetry) with Microsoft Sentinel (security analytics), or Defender for Cloud (posture and protection) with Sentinel (SIEM).

## Check yourself

1. Which service would you choose for a global HTTP entry point with WAF? For internal TCP load balancing?
2. A team wants to run containers without managing Kubernetes. Which service?
3. Match each to a messaging service: reliable ordered commands; high-volume telemetry; reacting to blob-created events.
4. Which database would you pick for globally distributed chat history? For migrating SQL Server with minimal change?
5. What were Microsoft Foundry and Foundry Tools previously called?
6. A client wants employees to ask questions over SharePoint content with no development. What do you suggest first?
7. What is the difference between Azure AI Search and Foundry IQ?
8. Name the services you would use for secrets, for observability and for security posture.

<details>
<summary>Answers</summary>

1. Azure Front Door. Azure Load Balancer (internal).
2. Azure Container Apps.
3. Service Bus; Event Hubs; Event Grid.
4. Azure Cosmos DB. Azure SQL Managed Instance.
5. Azure AI Foundry (and before that Azure AI Studio). Azure AI services (and before that Cognitive Services).
6. Microsoft 365 Copilot, after checking that SharePoint permissions are sound.
7. Azure AI Search is the search and retrieval engine. Foundry IQ is a knowledge layer for agents built on it that unifies multiple knowledge sources behind one retrieval endpoint.
8. Azure Key Vault (with managed identities); Azure Monitor and Application Insights; Microsoft Defender for Cloud.

</details>

## References

- [Azure products directory](https://azure.microsoft.com/products/)
- [Azure documentation](https://learn.microsoft.com/azure/)
- [Azure Architecture Center: technology choices](https://learn.microsoft.com/azure/architecture/guide/technology-choices/technology-choices-overview)
- [Choose an Azure compute service](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Load-balancing options](https://learn.microsoft.com/azure/architecture/guide/technology-choices/load-balancing-overview)
- [Choose between Azure messaging services](https://learn.microsoft.com/azure/service-bus-messaging/compare-messaging-services)
- [Understand data store models](https://learn.microsoft.com/azure/architecture/guide/technology-choices/data-store-overview)
- [What is Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/what-is-foundry)
- [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/)
