# Azure Roadmap

The complete topic tree for this repository. Snapshot: **October 2026**.

**How to read this file**

- Every `##` heading is a **topic** and becomes a **folder** (the slug is shown under the heading).
- Every `###` heading is a **subtopic** and becomes one **`.md` file** inside that folder.
- The line under each subtopic is its scope: what the file must cover.
- Track tags on each topic show who needs it: **E** = AI Engineer, **A** = AI Solution Architect, **C** = Consultant. `●` = core, `○` = working knowledge, `–` = awareness only.
- Names follow current Microsoft branding (Microsoft Foundry, Foundry Tools, Microsoft Entra). Former names are given in brackets where people still search for them.

**Contents**

| Part | Topics | Theme |
|---|---|---|
| A | 01 – 04 | Foundations |
| B | 05 – 11 | Core infrastructure and application platform |
| C | 12 | Data and analytics |
| D | 13 – 24 | AI: Foundry, models, RAG, agents, MLOps, Copilot |
| E | 25 – 32 | Secure, operate, migrate, extend |
| F | 33 – 38 | Architecture, consulting, certification, capstones |

---

# Part A · Foundations

## 01 · Cloud and Azure Fundamentals

`01-cloud-and-azure-fundamentals/` · E ● A ● C ●

### 01 · Cloud Computing Concepts
IaaS, PaaS, SaaS, serverless; public, private, hybrid; CapEx vs OpEx; elasticity and the consumption model.

### 02 · Shared Responsibility Model
Who owns what per service model, and how it shifts again for AI workloads.

### 03 · Azure Global Infrastructure
Geographies, regions, region pairs, availability zones, edge locations, sovereign regions.

### 04 · Azure Resource Hierarchy
Tenants, management groups, subscriptions, resource groups, resources.

### 05 · Azure Resource Manager
Control plane vs data plane, resource providers, API versions, locks, tags, idempotent deployments.

### 06 · Accounts, Subscriptions and Offers
Free account, pay-as-you-go, Enterprise Agreement, Microsoft Customer Agreement, CSP, dev/test offers.

### 07 · Portal, Cloud Shell and Mobile App
Navigating the portal, dashboards, Resource Graph Explorer, Cloud Shell.

### 08 · Service Lifecycle, SLAs and Quotas
Preview vs GA, retirement notices, composite SLA maths, quotas and limits, capacity requests.

### 09 · Azure Service Map
One-page orientation of every service category and when each one is the right choice.

## 02 · Identity and Access with Microsoft Entra

`02-identity-and-access/` · E ● A ● C ●

### 01 · Microsoft Entra ID Fundamentals
Tenants, users, groups, licences, directory roles, custom domains.

### 02 · Authentication
MFA, passwordless, passkeys, self-service password reset, authentication strengths.

### 03 · Azure RBAC
Role definitions, assignments, scopes, custom roles, deny assignments, ABAC conditions.

### 04 · Managed Identities and Workload Identity
System and user assigned identities, federated credentials, keyless access between services.

### 05 · App Registrations and Service Principals
OAuth 2.0, OpenID Connect, MSAL, scopes and app roles, on-behalf-of flow, certificates vs secrets.

### 06 · Conditional Access and Identity Protection
Policy design, risk signals, device compliance, Zero Trust enforcement.

### 07 · Privileged Identity Management and Identity Governance
Just-in-time elevation, access reviews, entitlement management, lifecycle workflows.

### 08 · Microsoft Entra External ID
Customer and partner identity, B2B collaboration, user flows (B2C is the legacy path).

### 09 · Hybrid Identity
Entra Connect and Cloud Sync, Entra Domain Services, Kerberos and legacy app access.

### 10 · Microsoft Entra Agent ID
Identities for AI agents: agent registry, lifecycle, access reviews and Conditional Access for agents.

### 11 · Entra Verified ID and Global Secure Access
Decentralised credentials, Entra Private Access and Internet Access.

## 03 · Governance and Landing Zones

`03-governance-and-landing-zones/` · E ○ A ● C ●

### 01 · Cloud Adoption Framework
Strategy, Plan, Ready, Adopt, Govern, Secure, Manage, plus the AI adoption scenario.

### 02 · Management Group and Subscription Design
Hierarchy patterns, subscription vending, subscription democratisation.

### 03 · Azure Landing Zones
Platform and workload landing zones, design principles and design areas, the accelerators (portal, Bicep, Terraform).

### 04 · AI Landing Zones
The AI application landing zone: where Foundry, search, gateways and data sit in an enterprise estate.

### 05 · Azure Policy
Definitions, initiatives, effects, remediation, exemptions, policy as code, built-in AI policies.

### 06 · Naming, Tagging and Resource Organisation
Conventions, tag inheritance, enforcing them with policy.

### 07 · Azure Resource Graph
KQL across the estate for inventory, compliance and change tracking.

### 08 · Regulatory Compliance
Compliance offerings, Service Trust Portal, data residency, EU Data Boundary.

### 09 · Azure Lighthouse and Managed Applications
Cross-tenant management for partners and service providers.

### 10 · Sovereign and Specialised Clouds
Azure Government, Azure operated by 21Vianet, Microsoft Sovereign Cloud options.

## 04 · Tooling and Infrastructure as Code

`04-tooling-and-infrastructure-as-code/` · E ● A ○ C ○

### 01 · Azure CLI
Install, authentication, output formats, JMESPath queries, extensions, scripting.

### 02 · Azure PowerShell
Az module, contexts, pipeline patterns, when to prefer it over the CLI.

### 03 · ARM Templates
Template anatomy, deployment scopes and modes; read-only knowledge for inherited estates.

### 04 · Bicep
Modules, parameter files, loops and conditions, `what-if`, registries, user-defined types.

### 05 · Terraform on Azure
AzureRM and AzAPI providers, remote state, workspaces, import and export.

### 06 · Azure Verified Modules
Consuming and composing the Microsoft-maintained Bicep and Terraform module library.

### 07 · Deployment Stacks and Template Specs
Lifecycle-managed deployments, deny settings, clean-up behaviour.

### 08 · Azure Developer CLI
`azd` templates, environments, hooks, provisioning and deploying AI apps end to end.

### 09 · Azure SDKs and REST API
SDK design guidelines, `DefaultAzureCredential`, retries, pagination, long-running operations.

### 10 · Azure MCP Server and AI-Assisted Operations
Driving Azure from coding agents: Azure MCP Server, GitHub Copilot for Azure, safe-use guardrails.

---

# Part B · Core Infrastructure and Application Platform

## 05 · Networking

`05-networking/` · E ○ A ● C ○

### 01 · Virtual Networks and Subnets
Address planning, subnet delegation, service endpoints, IPv6.

### 02 · Network Security Groups and Application Security Groups
Rule evaluation, service tags, flow logs.

### 03 · Routing
System routes, user-defined routes, BGP, Azure Route Server.

### 04 · VNet Peering and Azure Virtual Network Manager
Hub-spoke and mesh topologies, connectivity and security admin configurations.

### 05 · Private Link and Private Endpoints
Private access to PaaS, DNS integration, Private Link Service.

### 06 · Azure DNS
Public and private zones, DNS Private Resolver, split-horizon design.

### 07 · Load Balancing Decision Guide
Load Balancer, Application Gateway, Front Door, Traffic Manager: which one and why.

### 08 · Application Gateway and Web Application Firewall
Layer 7 routing, TLS termination, WAF policies, Application Gateway for Containers.

### 09 · Azure Front Door and CDN
Global entry point, caching, rules engine, Private Link origins.

### 10 · Azure Firewall and Firewall Manager
SKUs, rule collections, TLS inspection, IDPS, forced tunnelling.

### 11 · NAT Gateway and Outbound Connectivity
SNAT exhaustion, default outbound access retirement, egress design.

### 12 · VPN Gateway and ExpressRoute
Site-to-site, point-to-site, ExpressRoute circuits, Global Reach, resilience tiers.

### 13 · Virtual WAN
Managed hubs, routing intent, secured virtual hub, SD-WAN integration.

### 14 · Azure Bastion and Secure Admin Access
Bastion SKUs, just-in-time VM access, privileged workstations.

### 15 · DDoS Protection
Network and IP protection tiers, telemetry, response.

### 16 · Network Security Perimeter
Logical isolation boundary for PaaS services, access rules, transition mode.

### 17 · Network Watcher and Troubleshooting
Connection Monitor, IP flow verify, packet capture, VNet flow logs, Traffic Analytics.

### 18 · Networking for AI Workloads
Private Foundry and Azure OpenAI, agent egress control, VNet injection, on-premises data access.

## 06 · Compute

`06-compute/` · E ○ A ● C ○

### 01 · Virtual Machines
Series and sizing, generations, images, extensions, cloud-init, Arm-based sizes.

### 02 · Availability Sets, Zones and Virtual Machine Scale Sets
Flexible orchestration, autoscale, rolling upgrades, zone balancing.

### 03 · Spot, Reserved Capacity and Compute Fleet
Eviction handling, capacity reservations, mixing pricing models.

### 04 · Managed Disks
Disk types, performance tiers, bursting, encryption, snapshots, shared disks.

### 05 · Images and VM Image Builder
Azure Compute Gallery, golden images, Azure Linux.

### 06 · Dedicated Host and Confidential Computing
Isolation, confidential VMs, enclaves, Azure Attestation.

### 07 · GPU and AI Infrastructure
GPU VM families, InfiniBand, quota strategy, when to self-host models vs use Foundry.

### 08 · Azure Batch
Pools, jobs, tasks, autoscale formulas, rendering and simulation workloads.

### 09 · High Performance Computing
CycleCloud, HPC VM series, Slurm, Managed Lustre.

### 10 · Azure Virtual Desktop and Windows 365
Host pools, FSLogix, scaling plans, AVD Hybrid, Windows 365 Cloud PCs; Dev Box (maintenance mode).

### 11 · Azure VMware Solution and Partner Clouds
AVS, Nutanix Cloud Clusters, Oracle Database@Azure, SAP on Azure.

### 12 · Compute Decision Guide
VMs vs App Service vs Functions vs Container Apps vs AKS, with decision tree.

## 07 · Containers and Kubernetes

`07-containers-and-kubernetes/` · E ● A ● C ○

### 01 · Container Fundamentals
Images, layers, registries, OCI, multi-stage builds.

### 02 · Azure Container Registry
Tiers, ACR Tasks, geo-replication, artifact streaming, content trust, private access.

### 03 · Azure Container Instances
Container groups, confidential containers, standby pools, burst scenarios.

### 04 · Azure Container Apps
Environments, revisions, KEDA scaling, Dapr, jobs, workload profiles.

### 05 · Container Apps for AI
Serverless GPUs, dynamic sessions (code interpreter sandboxes), hosting agents and MCP servers.

### 06 · AKS Fundamentals
Control plane, node pools, AKS Automatic vs Standard, upgrades and support policy.

### 07 · AKS Networking
Azure CNI Overlay, Cilium, ingress and Gateway API, application routing, Istio add-on.

### 08 · AKS Security and Identity
Entra integration, Workload Identity, Azure Policy, Defender for Containers, image integrity.

### 09 · AKS Scaling and Cost
Cluster autoscaler, node auto-provisioning, KEDA, spot node pools, cost analysis.

### 10 · AKS for AI Workloads
KAITO (AI toolchain operator), GPU node pools, Ray, serving open models.

### 11 · AKS Fleet Manager and GitOps
Multi-cluster updates, workload placement, Flux and Argo CD.

### 12 · Azure Red Hat OpenShift and Service Fabric
Where they fit, and migration considerations.

### 13 · Azure Container Storage
Persistent storage for stateful containers.

## 08 · Storage

`08-storage/` · E ● A ● C ○

### 01 · Storage Accounts
Account kinds, redundancy (LRS, ZRS, GRS, GZRS), performance tiers, failover.

### 02 · Blob Storage
Containers, access tiers, lifecycle management, versioning, immutability, object replication.

### 03 · Azure Data Lake Storage
Hierarchical namespace, ACLs, lake layout for analytics and AI.

### 04 · Azure Files and File Sync
SMB and NFS shares, identity-based access, tiers, hybrid sync.

### 05 · Queue and Table Storage
Simple messaging and key-value storage, and when to use Service Bus or Cosmos DB instead.

### 06 · Storage Security
Entra authorisation, SAS, customer-managed keys, firewalls, private endpoints, Defender for Storage.

### 07 · Azure NetApp Files and Elastic SAN
Enterprise file and block storage for demanding workloads.

### 08 · Azure Managed Lustre
Parallel file system for HPC and model training.

### 09 · Storage Actions, Discovery and Insights
Serverless data management at scale, estate-wide visibility.

### 10 · Data Transfer
AzCopy, Storage Explorer, Storage Mover, Data Box.

### 11 · Storage for AI Workloads
Layout for RAG corpora, training data, checkpoints, and Foundry-attached storage.

## 09 · Databases

`09-databases/` · E ● A ● C ○

### 01 · Database Decision Guide
Relational vs document vs key-value vs graph vs vector; a selection matrix.

### 02 · Azure SQL Database
Purchasing models, serverless, Hyperscale, elastic pools, geo-replication.

### 03 · Azure SQL Managed Instance and SQL Server on VMs
Lift-and-shift paths, feature parity, licensing with Azure Hybrid Benefit.

### 04 · Azure Database for PostgreSQL
Flexible server, high availability, read replicas, extensions, Azure HorizonDB (preview).

### 05 · Azure Database for MySQL
Flexible server, HA, migration paths.

### 06 · Azure Cosmos DB
APIs, partitioning, consistency levels, request units, global distribution, change feed.

### 07 · Azure DocumentDB
MongoDB-compatible managed service (formerly Cosmos DB for MongoDB vCore).

### 08 · Azure Managed Redis
Tiers, clustering, persistence, semantic cache; migration from Azure Cache for Redis.

### 09 · Vector Search in Azure Databases
Vectors in SQL, PostgreSQL (pgvector, DiskANN), Cosmos DB and DocumentDB; when to use these vs Azure AI Search.

### 10 · Databases as Agent Memory and State
Conversation history, checkpoints, long-term memory patterns.

### 11 · Database Security and Compliance
Entra authentication, TDE, Always Encrypted, auditing, ledger, confidential ledger.

### 12 · Backup, HA and DR for Databases
Point-in-time restore, failover groups, RPO and RTO design.

### 13 · Database Migration
Database Migration Service, assessment tooling, cutover strategies.

## 10 · Application Platform

`10-application-platform/` · E ● A ● C ○

### 01 · Azure App Service
Plans, deployment slots, scaling, custom domains and certificates, managed instance option.

### 02 · App Service Networking and Security
VNet integration, private endpoints, Easy Auth, App Service Environment.

### 03 · Azure Functions
Triggers and bindings, hosting plans (Flex Consumption, Premium), cold start, scaling.

### 04 · Durable Functions and Durable Task Scheduler
Orchestrations, entities, long-running and human-in-the-loop workflows.

### 05 · Functions for AI
Azure OpenAI and MCP bindings, remote MCP servers on Functions, agent tool back ends.

### 06 · Static Web Apps
Front-end hosting, linked APIs, authentication, preview environments.

### 07 · Azure App Configuration
Feature flags, dynamic configuration, Key Vault references, AI configuration.

### 08 · Azure Communication Services
Chat, voice, video, SMS, email; voice agent integration.

### 09 · Azure SignalR, Web PubSub and Fluid Relay
Real-time messaging and streaming model output to clients.

### 10 · Azure Maps and Notification Hubs
Geospatial and push notification services.

### 11 · Application Modernisation Patterns
Strangler fig, twelve-factor, replatform vs refactor, AI-assisted modernisation.

## 11 · Integration, Messaging and APIs

`11-integration-messaging-and-apis/` · E ● A ● C ○

### 01 · Messaging Decision Guide
Event Grid vs Event Hubs vs Service Bus vs Storage Queues.

### 02 · Azure Service Bus
Queues, topics, sessions, dead-lettering, transactions, geo-replication.

### 03 · Azure Event Grid
Topics, namespaces, MQTT broker, CloudEvents, push and pull delivery.

### 04 · Azure Event Hubs
Partitions, consumer groups, Kafka endpoint, capture, schema registry.

### 05 · Azure Logic Apps
Consumption vs Standard, connectors, B2B, agent loops and AI workflows.

### 06 · Azure API Management
Tiers, policies, products and subscriptions, developer portal, workspaces, self-hosted gateway.

### 07 · API Management as AI Gateway
Token limits, load balancing and circuit breaking across model back ends, semantic caching, exposing and governing MCP servers.

### 08 · Azure API Center
API and MCP server inventory, governance, discovery.

### 09 · Azure Data Factory for Integration
Pipelines, integration runtimes, connectors (covered in depth in topic 12).

### 10 · Integration Architecture Patterns
Event-driven design, saga, outbox, claim check, competing consumers.

---

# Part C · Data and Analytics

## 12 · Data and Analytics

`12-data-and-analytics/` · E ○ A ● C ●

### 01 · Modern Data Architecture
Lakehouse, medallion layers, data mesh, Delta and Iceberg formats.

### 02 · Microsoft Fabric Fundamentals
Capacities, workspaces, OneLake, shortcuts, mirroring, licensing.

### 03 · Fabric Data Engineering and Data Factory
Lakehouse, Spark notebooks, pipelines, Dataflows Gen2.

### 04 · Fabric Data Warehouse and Databases
Warehouse, SQL database in Fabric, Cosmos DB in Fabric.

### 05 · Fabric Real-Time Intelligence
Eventstreams, Eventhouse, KQL, Activator.

### 06 · Power BI and Semantic Models
Direct Lake, modelling, DAX basics, Power BI Embedded, Copilot in Power BI.

### 07 · Fabric IQ and Data Agents
Ontology, semantic models as agent knowledge, Fabric data agents as a Foundry source.

### 08 · Azure Databricks
Workspaces, Unity Catalog, Delta Live Tables, Mosaic AI, integration with Foundry.

### 09 · Azure Data Factory and Azure Synapse Analytics
Existing estates, and how to plan migration to Fabric.

### 10 · Azure Data Explorer and Stream Analytics
Telemetry analytics and stream processing.

### 11 · Microsoft Purview Data Governance
Unified catalogue, lineage, classification, data quality, data products.

### 12 · Data Platform for AI Readiness
Getting data fit for RAG, fine-tuning and agents: quality, access control, freshness.

---

# Part D · AI on Azure

## 13 · AI and Generative AI Foundations

`13-ai-and-generative-ai-foundations/` · E ● A ● C ●

### 01 · Machine Learning Essentials
Supervised and unsupervised learning, training vs inference, evaluation metrics.

### 02 · Deep Learning and Transformers
Attention, encoder and decoder architectures, why scale matters.

### 03 · Large Language Models
Tokens, context windows, sampling parameters, reasoning models, limitations.

### 04 · Embeddings and Vector Similarity
What embeddings capture, distance metrics, dimensionality, indexing algorithms.

### 05 · Multimodal Models
Vision, audio, image and video generation, real-time speech models.

### 06 · Small Language Models and Open Models
Phi family, open-weight models, on-device inference, trade-offs.

### 07 · AI Solution Patterns
Chat, RAG, extraction, classification, copilots, single agents, multi-agent systems.

### 08 · The Microsoft AI Stack
How Foundry, Copilot Studio, Microsoft 365 Copilot, Fabric and Azure ML fit together.

### 09 · AI Use Case Qualification
Is this a good AI problem? Feasibility, value, risk and data-readiness checks.

## 14 · Microsoft Foundry Platform

`14-microsoft-foundry-platform/` · E ● A ● C ●

### 01 · Microsoft Foundry Overview
What it is, the rename from Azure AI Foundry and Azure AI Studio, portal tour.

### 02 · Foundry Resources and Projects
Foundry resource, projects, the older hub-based model, migration between them.

### 03 · Foundry Access Control
Built-in roles, project isolation, managed identities, keyless authentication.

### 04 · Foundry Networking
Private endpoints, managed network, network-isolated agent setup, egress rules.

### 05 · Connections and Bring-Your-Own Resources
Connecting storage, search, Cosmos DB, Key Vault and external services.

### 06 · Foundry SDKs and APIs
Foundry SDK, OpenAI-compatible endpoints, Responses API, language support.

### 07 · Foundry Toolkit for VS Code and Developer Workflow
Local build, debug and deploy loop, `azd` templates, playgrounds.

### 08 · Foundry Local and Edge Inference
Running models on device, Windows integration, hybrid cloud and local patterns.

### 09 · Foundry Control Plane
Fleet-wide inventory, policy, guardrails and monitoring for agents and models.

### 10 · Quotas, Regions and Pricing
Regional availability, quota management, pricing meters, cost attribution per project.

### 11 · Foundry Infrastructure as Code
Bicep and Terraform for Foundry resources, projects, deployments and connections.

## 15 · Foundry Models and Azure OpenAI

`15-foundry-models-and-azure-openai/` · E ● A ● C ○

### 01 · Model Catalogue
Models sold directly by Azure vs partner and community models; licensing and support differences.

### 02 · Azure OpenAI in Foundry Models
GPT, reasoning, image, audio and video model families; API versions and the v1 API.

### 03 · Other Model Families
Anthropic Claude, Microsoft MAI, Phi, Meta, Mistral, xAI, DeepSeek, Cohere, NVIDIA, Fireworks AI on Foundry.

### 04 · Deployment Types
Global, Data Zone and Regional; Standard, Provisioned and Batch; data residency implications.

### 05 · Provisioned Throughput
PTU sizing, reservations, spillover, when it beats pay-as-you-go.

### 06 · Model Router and Model Selection
Automatic routing, benchmarking, leaderboards, choosing on quality, latency and cost.

### 07 · Chat Completions and Responses API
Messages, streaming, stateful responses, background mode.

### 08 · Function Calling and Structured Outputs
Tool definitions, JSON schema, parallel calls, validation.

### 09 · Embedding Models
Model choice, dimensions, batching, cost.

### 10 · Vision, Image and Video Generation
Image understanding, image generation and editing, video generation.

### 11 · Audio and Real-Time APIs
Speech in and out, real-time WebRTC and WebSocket sessions.

### 12 · Batch, Prompt Caching and Cost Optimisation
Batch API, cached tokens, priority processing, token budgeting.

### 13 · Rate Limits, Quotas and Resilience
TPM and RPM, retries and backoff, multi-region back ends.

### 14 · Model Lifecycle
Versions, auto-upgrade policy, deprecation and retirement planning.

### 15 · Managed Compute and Self-Hosted Deployment
Deploying open models on dedicated compute, GPU capacity, when to self-host.

## 16 · Prompt Engineering and Model Customisation

`16-prompt-engineering-and-model-customisation/` · E ● A ○ C ○

### 01 · Prompt Design Fundamentals
Instructions, context, examples, output format, system messages.

### 02 · Prompting Reasoning Models
Effort settings, when to let the model plan, what not to over-specify.

### 03 · Context Engineering
Deciding what goes in the window: retrieval, memory, tool results, compaction.

### 04 · Prompt Management
Versioning, templates, prompt-as-code, regression testing.

### 05 · Customisation Decision Guide
Prompting vs RAG vs fine-tuning vs distillation.

### 06 · Fine-Tuning in Foundry
Supervised, reinforcement and vision fine-tuning; data preparation; developer tier; Frontier Tuning.

### 07 · Distillation and Synthetic Data
Stored completions, teacher and student models, generating training data.

### 08 · Deploying and Operating Fine-Tuned Models
Hosting cost, evaluation before promotion, retraining triggers.

## 17 · Retrieval-Augmented Generation and Knowledge

`17-rag-and-knowledge/` · E ● A ● C ○

### 01 · RAG Fundamentals
Why RAG, the pipeline end to end, common failure modes.

### 02 · Azure AI Search Fundamentals
Service tiers, indexes, fields, analysers, capacity planning.

### 03 · Ingestion: Indexers and Skillsets
Data sources, change tracking, enrichment, integrated vectorisation.

### 04 · Document Processing and Chunking
Layout-aware parsing, chunk strategies, tables and images, metadata.

### 05 · Vector Search
HNSW vs exhaustive KNN, quantisation, filters, multi-vector fields.

### 06 · Hybrid Search and Semantic Ranker
Reciprocal rank fusion, reranking, scoring profiles, query rewriting.

### 07 · Agentic Retrieval
Query planning, parallel sub-queries, knowledge bases and knowledge sources in Azure AI Search.

### 08 · Foundry IQ
Unified knowledge layer for agents: knowledge bases, sources (SharePoint, OneLake, Azure SQL, web, MCP), serverless tier.

### 09 · Work IQ, Fabric IQ and Web Grounding
Grounding on Microsoft 365 data, business data and live web content.

### 10 · Security Trimming and Permissions
Document-level access control, ACL and sensitivity-label propagation, per-user retrieval.

### 11 · Multimodal and Structured-Data RAG
Images and diagrams, text-to-SQL, graph-based retrieval.

### 12 · RAG Evaluation and Tuning
Retrieval metrics, groundedness, golden datasets, iteration loop.

### 13 · RAG at Scale
Multi-tenancy, index partitioning, freshness, cost and latency trade-offs.

## 18 · Agents with Foundry Agent Service

`18-agents-with-foundry-agent-service/` · E ● A ● C ●

### 01 · Agent Fundamentals
Model, instructions, tools, memory, the agent loop; when not to build an agent.

### 02 · Foundry Agent Service Overview
Agent types (prompt, workflow, hosted), setups (basic, standard, network-isolated).

### 03 · Prompt Agents
Declarative agents in the portal and SDK, versioning, testing.

### 04 · Hosted Agents
Bring your own code and framework, sandboxed sessions, Responses and Invocations protocols.

### 05 · Built-In Tools
File search, code interpreter, web and Bing grounding, Azure AI Search, deep research, browser automation, computer use.

### 06 · Custom Tools
Function tools, OpenAPI tools, Azure Functions, Logic Apps connectors.

### 07 · MCP Tools, Toolboxes and Skills
Connecting MCP servers, the tool catalogue, Toolboxes, skills, tool search.

### 08 · Agent Memory
Session, user and procedural memory; bring-your-own thread storage.

### 09 · Multi-Agent Workflows
Connected agents, workflow orchestration, human-in-the-loop, handoffs.

### 10 · Routines and Long-Running Agents
Scheduled and event-triggered runs, background execution.

### 11 · Voice Agents
Voice Live API, real-time speech agents, avatars, telephony integration.

### 12 · Publishing Agents
Teams, Microsoft 365 Copilot, web apps, APIs and A2A endpoints.

### 13 · Agent Identity and Authorisation
Entra Agent ID, on-behalf-of access, least privilege for tools.

### 14 · Agent Design Patterns and Anti-Patterns
Single vs multi-agent, tool design, failure handling, cost control.

## 19 · Agent Frameworks and Protocols

`19-agent-frameworks-and-protocols/` · E ● A ○ C –

### 01 · Microsoft Agent Framework
Agents, the harness, middleware, sessions; Python and .NET.

### 02 · Agent Framework Workflows
Graph-based orchestration, checkpoints, sequential, concurrent, handoff and Magentic patterns.

### 03 · Migrating from Semantic Kernel and AutoGen
What changed, mapping concepts, migration steps.

### 04 · Model Context Protocol
Servers, clients, tools, resources and prompts; transports; authorisation.

### 05 · Building and Hosting MCP Servers on Azure
Functions, Container Apps, API Management; securing and governing them.

### 06 · Agent-to-Agent Protocol
Agent cards, task lifecycle, cross-platform interoperability.

### 07 · Third-Party Frameworks on Foundry
LangChain and LangGraph, CrewAI, LlamaIndex, OpenAI Agents SDK, Claude Agent SDK.

### 08 · Microsoft 365 Agents SDK and Toolkit
Building agents for Teams and Microsoft 365 channels in code.

### 09 · Framework Selection Guide
Which framework for which situation, and when none is needed.

## 20 · Foundry Tools

`20-foundry-tools/` · E ● A ○ C ○

Formerly Azure AI services and, before that, Cognitive Services.

### 01 · Foundry Tools Overview
Resource types, multi-service resource, authentication, containers, limits.

### 02 · Azure Vision
Image analysis, OCR, Face, spatial analysis, Custom Vision status.

### 03 · Azure Speech
Speech to text, text to speech, custom and neural voices, speech translation, pronunciation assessment.

### 04 · Azure Language
Entity recognition, PII detection, summarisation, conversational language understanding, question answering.

### 05 · Azure Translator
Text and document translation, custom translator.

### 06 · Azure Document Intelligence
Prebuilt, layout and custom models; batch processing.

### 07 · Azure Content Understanding
Multimodal extraction from documents, images, audio and video with analysers.

### 08 · Azure AI Video Indexer
Video and audio insights, Arc-enabled deployment.

### 09 · Azure AI Content Safety
Harm categories, Prompt Shields, groundedness detection, protected material, custom categories.

### 10 · Bot Service, Health Bot and Immersive Reader
Channel connectivity and specialised services.

### 11 · Foundry Tools vs LLM Decision Guide
When a task-specific service beats a general model on cost, accuracy and compliance.

## 21 · Azure Machine Learning and MLOps

`21-azure-machine-learning-and-mlops/` · E ○ A ○ C –

### 01 · Azure Machine Learning Overview
Workspaces, registries, how it relates to Foundry.

### 02 · Compute
Compute instances, clusters, serverless compute, Kubernetes attach.

### 03 · Data and Feature Management
Data assets, datastores, managed feature store.

### 04 · Training
Jobs, environments, distributed training, hyperparameter sweeps.

### 05 · Automated ML
Tabular, vision and NLP tasks; when it is good enough.

### 06 · Pipelines and Components
Reusable components, scheduling, lineage.

### 07 · Model Management and MLflow
Tracking, model registry, packaging.

### 08 · Deployment
Managed online endpoints, batch endpoints, safe rollout.

### 09 · MLOps
CI/CD for models, monitoring, drift detection, retraining.

### 10 · Responsible AI Dashboard
Error analysis, interpretability, fairness assessment.

## 22 · Evaluation, Observability and GenAIOps

`22-evaluation-observability-and-genaiops/` · E ● A ● C ○

### 01 · GenAIOps Lifecycle
From experiment to production: stages, roles, artefacts.

### 02 · Evaluation Fundamentals
Offline vs online, datasets, human vs automated judging.

### 03 · Built-In Evaluators
Quality, groundedness, safety, agent evaluators (task adherence, tool accuracy, intent resolution).

### 04 · Custom and Rubric Evaluators
Code-based graders, LLM-as-judge, rubric design, calibration.

### 05 · Evaluation in CI/CD
GitHub Actions and Azure DevOps gates, regression thresholds.

### 06 · Tracing with OpenTelemetry
GenAI semantic conventions, tracing any framework, Application Insights integration.

### 07 · Continuous Evaluation and Monitoring
Production sampling, dashboards, alerts, cost and latency tracking.

### 08 · Agent Optimisation
Using production traces to improve prompts, tools and models.

### 09 · AI Red Teaming
AI Red Teaming Agent, PyRIT, attack strategies, reporting.

### 10 · Measuring Business Value
Task completion, time saved, adoption, agent ROI.

## 23 · Responsible AI, AI Security and Governance

`23-responsible-ai-security-and-governance/` · E ● A ● C ●

### 01 · Responsible AI Principles
Microsoft Responsible AI Standard, impact assessments, transparency notes.

### 02 · AI Threat Landscape
Direct and indirect prompt injection, data exfiltration, tool abuse, OWASP Top 10 for LLMs and agents.

### 03 · Guardrails and Content Filters
Filter configuration, blocklists, guardrails at input, output and tool-call points.

### 04 · Securing Agents
Least-privilege tools, approval gates, sandboxing, agent control specifications.

### 05 · Data Protection for AI
Microsoft Purview for AI, DSPM for AI, sensitivity labels, DLP, oversharing controls.

### 06 · Defender for Cloud AI Protection
AI security posture management, runtime threat protection for AI services.

### 07 · Data Privacy, Residency and Abuse Monitoring
How customer data is handled, zero data retention options, regional processing.

### 08 · AI Regulation and Standards
EU AI Act, NIST AI RMF, ISO/IEC 42001; mapping obligations to controls.

### 09 · AI Governance Operating Model
Review boards, intake process, model and agent inventory, policy enforcement.

### 10 · Agent Governance at Scale
Agent 365, Foundry Control Plane and Entra Agent ID working together; controlling agent sprawl.

## 24 · Copilot, Low-Code Agents and Azure AI Assistants

`24-copilot-and-low-code-agents/` · E ○ A ● C ●

### 01 · Microsoft 365 Copilot Architecture
Microsoft Graph grounding, semantic index, security model, licensing.

### 02 · Extending Microsoft 365 Copilot
Declarative agents, custom engine agents, connectors, API plugins.

### 03 · Copilot Studio
Topics, generative orchestration, knowledge, actions, autonomous agents, MCP.

### 04 · Copilot Studio with Foundry
Bring your own model, calling Foundry agents, pro-code and low-code together.

### 05 · Build vs Buy Decision Guide
Microsoft 365 Copilot vs Copilot Studio vs Foundry vs custom code.

### 06 · Agent 365
The enterprise control plane for agents: registry, access, security, licensing.

### 07 · GitHub Copilot for Azure Workloads
Agent mode, coding agent, app modernisation, custom instructions.

### 08 · Azure Copilot
Agentic assistance in the portal for operations, migration, cost and troubleshooting.

### 09 · Azure SRE Agent
Automated incident investigation and remediation.

### 10 · Microsoft Security Copilot
Security operations assistance and agents.

### 11 · Power Platform and Dynamics 365 AI
Where low-code apps and business applications meet Azure AI.

---

# Part E · Secure, Operate, Migrate and Extend

## 25 · Security

`25-security/` · E ○ A ● C ●

### 01 · Zero Trust Architecture
Principles, pillars, mapping to Azure controls.

### 02 · Microsoft Cloud Security Benchmark
Control families, using it as a baseline, Secure Score.

### 03 · Azure Key Vault
Secrets, keys, certificates, RBAC, rotation, Managed HSM, Cloud HSM.

### 04 · Encryption
At rest, in transit, in use; customer-managed keys; double encryption.

### 05 · Microsoft Defender for Cloud
CSPM, workload protection plans, attack path analysis, multicloud coverage.

### 06 · Microsoft Sentinel
Data connectors, analytics rules, hunting, automation, data lake.

### 07 · Network Security Design
Segmentation, inspection, private access, egress control.

### 08 · Application and API Security
Authentication patterns, secrets hygiene, WAF, API protection.

### 09 · DevSecOps and Supply Chain
GitHub Advanced Security, code and secret scanning, Artifact Signing, SBOM.

### 10 · Microsoft Purview Compliance
Information protection, DLP, insider risk, audit, eDiscovery.

### 11 · Defender External Attack Surface Management
Discovering and monitoring internet-exposed assets.

### 12 · Incident Response on Azure
Detection, containment, forensics, post-incident review.

## 26 · Monitoring and Observability

`26-monitoring-and-observability/` · E ● A ● C ○

### 01 · Azure Monitor Overview
Data platform, metrics vs logs vs traces, data collection rules.

### 02 · Log Analytics and KQL
Workspace design, table plans, retention, essential KQL.

### 03 · Application Insights
OpenTelemetry instrumentation, distributed tracing, application map, availability tests.

### 04 · Alerts and Action Groups
Metric, log and activity alerts; dynamic thresholds; alert processing rules.

### 05 · Workbooks and Dashboards
Building operational views, Azure Monitor dashboards with Grafana.

### 06 · Container and Kubernetes Monitoring
Container insights, managed Prometheus, Managed Grafana.

### 07 · VM and Network Monitoring
VM insights, Azure Monitor Agent, network insights.

### 08 · Service Health, Resource Health and Advisor
Platform incidents, planned maintenance, recommendations.

### 09 · Monitoring AI Workloads
Token usage, latency, throttling, content filter events, agent traces.

### 10 · Observability Cost Control
Ingestion planning, sampling, basic and auxiliary logs, commitment tiers.

## 27 · DevOps and Platform Engineering

`27-devops-and-platform-engineering/` · E ● A ○ C ○

### 01 · Git and Branching Strategy
Trunk-based development, pull requests, protection rules.

### 02 · GitHub Actions for Azure
Workflows, OIDC federation, environments, reusable workflows.

### 03 · Azure DevOps
Boards, Repos, Pipelines, Artifacts, Test Plans, Managed DevOps Pools.

### 04 · CI/CD for Infrastructure
Bicep and Terraform pipelines, `what-if` and plan gates, drift detection.

### 05 · CI/CD for Applications
Build, test, containerise, deploy to App Service, Container Apps and AKS.

### 06 · CI/CD for AI Applications and Agents
Prompt and agent versioning, evaluation gates, environment promotion.

### 07 · Release Strategies
Blue-green, canary, feature flags, progressive exposure.

### 08 · Testing
Azure App Testing (load testing, Playwright workspaces), Chaos Studio.

### 09 · Developer Platforms
Windows 365 developer Cloud PCs (Dev Box is in maintenance mode), Deployment Environments, DevTest Labs, golden paths.

### 10 · Platform Engineering
Internal developer platforms, self-service, paved roads, measuring developer experience.

## 28 · Resiliency, Backup and Disaster Recovery

`28-resiliency-backup-and-disaster-recovery/` · E ○ A ● C ○

### 01 · Reliability Fundamentals
RTO, RPO, SLOs, failure mode analysis, redundancy vs recoverability.

### 02 · Availability Zones and Multi-Region Design
Zone-redundant vs zonal, active-active vs active-passive, region selection.

### 03 · Azure Backup
Vaults, policies, immutability, soft delete, cross-region restore.

### 04 · Azure Site Recovery
Replication, recovery plans, test failover.

### 05 · Data Tier Resilience
Storage, SQL, Cosmos DB and PostgreSQL replication and failover options.

### 06 · Resilience for AI Workloads
Multi-region model deployments, gateway failover, index and agent state recovery.

### 07 · Chaos Engineering and Resiliency Tooling
Chaos Studio experiments, Azure Infrastructure Resiliency Manager, resilience reviews.

### 08 · Business Continuity Planning
Runbooks, drills, dependency mapping, communication plans.

## 29 · Cost Management and FinOps

`29-cost-management-and-finops/` · E ○ A ● C ●

### 01 · Azure Pricing Fundamentals
Meters, billing scopes, invoices, the pricing calculator.

### 02 · Microsoft Cost Management
Cost analysis, budgets, alerts, exports, anomaly detection.

### 03 · Commitment Discounts
Reservations, savings plans, Azure Hybrid Benefit, MACC.

### 04 · Cost Allocation
Tagging strategy, showback and chargeback, shared cost splitting.

### 05 · Optimisation Techniques
Rightsizing, autoscale, spot, storage tiering, Advisor recommendations.

### 06 · AI Cost Modelling
Token economics, PTU vs pay-as-you-go break-even, search and agent run costs, caching.

### 07 · FinOps Framework
Inform, Optimise, Operate; maturity model; FinOps toolkit and hubs.

### 08 · Estimating a Solution
Building a defensible estimate and TCO for a proposal.

## 30 · Migration and Modernisation

`30-migration-and-modernisation/` · E – A ● C ●

### 01 · Migration Strategy
The Rs (rehost, replatform, refactor, rearchitect, rebuild, replace, retire), wave planning.

### 02 · Azure Migrate
Discovery, dependency analysis, assessment, business case, agentic migration.

### 03 · Server and VMware Migration
Replication, cutover, Azure VMware Solution path.

### 04 · Database Migration
SQL Server, Oracle, PostgreSQL and MySQL paths and tooling.

### 05 · Application Modernisation
App Service and Container Apps migration, GitHub Copilot app modernisation for .NET and Java.

### 06 · Data and Analytics Migration
Moving warehouses and pipelines to Fabric or Databricks.

### 07 · Migrating AI Workloads to Foundry
From hubs, Assistants API, Semantic Kernel, prompt flow and other clouds.

### 08 · Post-Migration Optimisation
Rightsizing, governance alignment, operational handover.

## 31 · Hybrid, Multicloud and Edge

`31-hybrid-multicloud-and-edge/` · E – A ● C ○

### 01 · Azure Arc Overview
Projecting external resources into Azure; the control plane model.

### 02 · Arc-Enabled Servers and Kubernetes
Onboarding, policy, monitoring, updates, GitOps.

### 03 · Arc-Enabled Data and Application Services
SQL Server enabled by Arc, data services, app services on Arc.

### 04 · Azure Local
Hyperconverged infrastructure (formerly Azure Stack HCI), AKS on Azure Local, disconnected operations.

### 05 · Azure Stack Hub and Edge Devices
Disconnected and rugged scenarios.

### 06 · Multicloud Management
Defender for Cloud and Arc across AWS and GCP, multicloud connectivity.

### 07 · AI at the Edge
Foundry Local, edge RAG, Arc-enabled AI services.

### 08 · Azure Operator Nexus and Telecom
Carrier-grade platform, awareness level.

## 32 · IoT, Industry and Specialised Services

`32-iot-industry-and-specialised-services/` · E – A ○ C ○

### 01 · Azure IoT Hub and Device Provisioning
Device identity, messaging, device twins, provisioning at scale.

### 02 · Azure IoT Operations and IoT Edge
Edge data plane on Arc-enabled Kubernetes, MQTT broker, connectors.

### 03 · Azure Digital Twins
Modelling environments, twin graphs, integration.

### 04 · Healthcare
Azure Health Data Services (FHIR, DICOM), healthcare agent service, healthcare AI models.

### 05 · Financial Services, Retail and Manufacturing
Industry reference patterns, data models and compliance concerns.

### 06 · Energy, Sustainability and Geospatial
Azure Data Manager for Energy, Planetary Computer Pro, emissions data.

### 07 · Azure Quantum
Awareness: hardware partners, hybrid quantum workflows.

### 08 · Microsoft Marketplace and Partner Solutions
Azure Native ISV services, private offers, transacting AI apps and agents.

---

# Part F · Architecture, Consulting and Career

## 33 · Solution Architecture

`33-solution-architecture/` · E ○ A ● C ●

### 01 · Azure Well-Architected Framework
Reliability, Security, Cost Optimisation, Operational Excellence, Performance Efficiency; trade-offs.

### 02 · Well-Architected Reviews and Service Guides
Running an assessment, using per-service guidance.

### 03 · Requirements and Constraints
Functional and non-functional requirements, quality attributes, assumptions.

### 04 · Cloud Design Patterns
Retry, circuit breaker, bulkhead, CQRS, cache-aside, gateway patterns.

### 05 · Architecture Styles
N-tier, microservices, event-driven, web-queue-worker, big data.

### 06 · Multitenant Architecture
Tenancy models, isolation, noisy neighbours, per-tenant cost.

### 07 · Mission-Critical Design
Deployment stamps, health modelling, zero-downtime deployment.

### 08 · Architecture Decision Records and Diagrams
Recording decisions, C4 model, Azure icon conventions.

### 09 · Azure Architecture Center
Finding and adapting reference architectures.

### 10 · Technology Decision Guides
Consolidated compute, data, messaging, load balancing and AI decision trees.

## 34 · AI Solution Architecture

`34-ai-solution-architecture/` · E ○ A ● C ●

### 01 · Well-Architected for AI Workloads
Design principles and design areas specific to AI.

### 02 · Baseline Foundry Chat Reference Architecture
The canonical secure, zonal chat application, component by component.

### 03 · Enterprise RAG Architecture
Ingestion, indexing, retrieval, orchestration and security trimming at scale.

### 04 · Agentic Architecture
Single agent, orchestrated multi-agent, workflow agents; choosing a topology.

### 05 · Generative AI Gateway Architecture
Central vs federated gateway, multi-back-end routing, chargeback.

### 06 · Network-Isolated AI Architecture
Fully private Foundry, search, storage and agents in a landing zone.

### 07 · Multi-Region and High-Availability AI
Capacity strategy, failover, data-zone constraints.

### 08 · Multitenant AI Solutions
Tenant isolation for models, indexes, memory and agents.

### 09 · Capacity and Performance Planning
Throughput modelling, latency budgets, load testing AI systems.

### 10 · Document and Voice Solution Architectures
Intelligent document processing, contact centre and voice agent designs.

### 11 · Integrating AI with Enterprise Systems
SAP, Dynamics 365, ServiceNow, Salesforce, line-of-business APIs.

### 12 · AI Platform Operating Model
AI centre of excellence, platform team vs workload teams, shared services.

## 35 · Consulting Practice

`35-consulting-practice/` · E – A ○ C ●

### 01 · The Azure Consultant Role
Engagement types, responsibilities, working with Microsoft and partners.

### 02 · Discovery and Envisioning Workshops
Stakeholder interviews, current-state assessment, use-case ideation.

### 03 · Use-Case Prioritisation and Business Case
Value vs feasibility scoring, ROI and TCO models, success metrics.

### 04 · AI Readiness Assessment
Data, platform, security, skills and governance maturity.

### 05 · Proof of Concept to Production
PoC and MVP scoping, exit criteria, avoiding pilot purgatory.

### 06 · Proposals, Estimation and Statements of Work
Scoping, effort estimates, assumptions, risks, deliverables.

### 07 · Solution Design Deliverables
High-level and low-level design documents, decision logs, handover packs.

### 08 · Stakeholder Communication
Executive storytelling, demos, handling objections, explaining AI risk plainly.

### 09 · Change Management and Adoption
Training, champions, usage measurement, adoption playbooks.

### 10 · Licensing and Commercials
Azure agreements, Copilot and Agent 365 licensing, partner programmes and funding.

### 11 · Delivery Governance
Agile delivery for AI projects, risk registers, quality gates.

## 36 · Certifications and Career

`36-certifications-and-career/` · E ● A ● C ●

Exam codes changed substantially in 2026. Confirm status on Microsoft Learn before booking.

### 01 · Certification Landscape
What retired, what replaced it, how renewals and applied skills work.

### 02 · AZ-900 Azure Fundamentals
Study plan mapped to topics 01 – 03.

### 03 · AI-901 Azure AI Fundamentals
Replacement for AI-900.

### 04 · AZ-104 Azure Administrator
Study plan mapped to topics 02 – 09 and 26.

### 05 · AI-103 Azure AI Apps and Agents Developer
Replacement for AI-102; mapped to topics 14 – 23.

### 06 · AI-200 Azure AI Cloud Developer
Replacement for AZ-204; mapped to topics 07 and 09 – 11.

### 07 · AI-300 MLOps Engineer
Replacement for DP-100; mapped to topics 21 – 22.

### 08 · AZ-305 Azure Solutions Architect Expert
Mapped to topics 03 – 12, 25 – 29 and 33.

### 09 · AB-100 Agentic AI Business Solutions Architect
Mapped to topics 18, 24, 34 and 35.

### 10 · Security, Data and DevOps Certifications
SC-500, SC-100, DP-700, DP-600, AZ-400, AZ-700.

### 11 · Portfolio, Interviews and Career Paths
What to build, how to present it, interview question bank per role.

## 37 · Capstone Projects

`37-capstone-projects/` · E ● A ● C ○

### 01 · Secure Landing Zone with IaC
Deploy a governed hub-spoke foundation with Bicep or Terraform and CI/CD.

### 02 · Enterprise RAG Assistant
Private, permission-aware RAG over mixed documents with evaluation.

### 03 · Multi-Agent Business Process
Orchestrated agents with tools, memory, approvals and tracing.

### 04 · Voice Agent
Real-time speech agent with telephony and tool calls.

### 05 · Intelligent Document Processing Pipeline
Extraction, validation, human review and downstream integration.

### 06 · Generative AI Gateway
API Management in front of multiple model back ends with quotas and chargeback.

### 07 · MCP Server on Azure
Build, secure, host and govern a remote MCP server.

### 08 · GenAIOps Pipeline
Evaluation gates, red teaming and continuous monitoring in CI/CD.

### 09 · Architecture Review Exercise
Run a Well-Architected review on a provided design and write the findings.

### 10 · Client Engagement Simulation
Discovery notes to proposal, design document, estimate and executive readout.

## 38 · Staying Current and Legacy Services

`38-staying-current-and-legacy-services/` · E ○ A ● C ●

### 01 · Tracking Azure Change
Azure Updates, Foundry what's new, Microsoft Learn release notes, Build and Ignite.

### 02 · Product Rename Map
Old name to current name for every renamed service (Azure AD, Cognitive Services, AI Studio and more).

### 03 · Retired and Retiring Services
Retirement dates and migration targets: Azure Cache for Redis, Spring Apps, MariaDB, Blueprints, Cloud Services, Media Services, Anomaly Detector, Azure API for FHIR and others.

### 04 · Legacy AI Stack
Assistants API, hub-based projects, prompt flow, Semantic Kernel and AutoGen: what to know when you inherit them.

### 05 · Legacy Data and Compute
Synapse, HDInsight, Data Lake Analytics, Analysis Services, classic resources.

### 06 · Preview Feature Policy
How to evaluate, adopt and communicate preview features to clients.

### 07 · Glossary
Every acronym and term used in this roadmap.
