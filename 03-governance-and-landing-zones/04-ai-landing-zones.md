# AI Landing Zones

## Why it matters

The first AI project in most organisations is a proof of concept in someone's subscription with public endpoints and an API key. The second is the same thing in a different subscription. By the fifth, security has stopped approving anything. An AI landing zone is the answer to "how do we let many teams build AI safely?": a governed place for AI workloads, with network, identity, policy and cost controls decided once. This is the file where the roadmap's foundations meet its AI focus.

## Key concepts

### An AI landing zone is a workload landing zone

There is no separate AI architecture. Cloud Adoption Framework guidance is explicit: if you have a platform landing zone, **deploy AI workloads to ordinary workload landing zones**, and add AI-specific governance through policy.

"AI landing zone" therefore means two things in practice:

1. **The pattern:** a workload landing zone configured for AI, with the networking, identity and policy that Foundry-based applications need.
2. **The accelerator:** Microsoft-published IaC (Bicep and Terraform, built on Azure Verified Modules) that deploys a production-ready baseline for Foundry workloads, in variants **with** and **without** an existing platform landing zone.

### Three AI governance decisions from CAF

| Decision | Guidance |
|---|---|
| **Separate internet-facing and internal AI workloads** | Use the *Online* and *Corp* management groups. The boundary stops external users reaching internal business data and lets policy differ |
| **Apply AI-specific policies per management group** | Start from the platform baseline, then add policy for Foundry models, Foundry Tools, Azure AI Search and GPU compute |
| **Deploy AI resources in workload subscriptions** | Not in platform subscriptions. Workload teams own their AI resources and inherit governance from their management group |

### Where things live

| Component | Location | Owner |
|---|---|---|
| Hub networking, firewall, private DNS zones, hybrid connectivity | Platform (connectivity) | Platform team |
| Central logging, Sentinel, Defender for Cloud | Platform (management, security) | Platform and security teams |
| **Shared AI gateway** (API Management) and shared model capacity, if offered | A shared AI platform subscription, usually under *Corp* | AI platform team |
| **Foundry resource and projects**, search, storage, databases, app hosting for a workload | The workload's subscriptions | Workload team |
| Agent and model inventory, cross-workload AI governance | Foundry Control Plane, Agent 365, Purview | AI governance function |

### Topologies for organising Foundry

How many Foundry resources, and who shares them, is the main structural choice.

| Topology | Description | Strengths | Weaknesses |
|---|---|---|---|
| **Per-workload** | Each workload has its own Foundry resource and projects in its own subscription | Clear isolation, ownership, cost and quota; simplest security story | Duplicated setup; quota fragmented across subscriptions |
| **Shared per business unit or domain** | One Foundry resource with a project per team or use case | Less duplication; shared connections and capacity | Coarser isolation; shared limits; needs careful RBAC |
| **Central AI platform with gateway** | Central team hosts model deployments behind API Management; workloads consume through the gateway and keep their own projects, data and agents | Central control of capacity, cost, safety policy and model lifecycle; chargeback | The gateway becomes a critical dependency and the platform team a potential bottleneck |

A Foundry **project** is the unit of isolation for a team or use case: its own agents, files, evaluations and access. A Foundry **resource** is the unit of networking, model deployments, policy and billing. Rules of thumb:

- Different data sensitivity or network exposure: **separate resources**.
- Production and non-production: **separate resources**, in separate subscriptions.
- Related use cases owned by one team at the same sensitivity: **projects in one resource**.

Most organisations end up with a hybrid: a central gateway and shared capacity for commonly used models, plus per-workload Foundry resources for agents, data and anything needing isolation. Topic 34 treats the gateway architecture in full.

### Reference components

A network-isolated AI workload landing zone typically contains:

| Layer | Components |
|---|---|
| **Ingress** | Application Gateway with WAF (regional) or Front Door (global); DDoS protection for internet-facing workloads |
| **Application** | Container Apps, App Service or AKS hosting the front end, API and orchestration |
| **AI** | Foundry resource and project; model deployments; Foundry Agent Service in a network-isolated (standard, bring-your-own-resources) setup |
| **Knowledge** | Azure AI Search; Storage account for source documents |
| **State** | Cosmos DB for conversation and agent thread storage |
| **Secrets** | Key Vault |
| **Gateway** | API Management as AI gateway (in the workload or shared) |
| **Networking** | Spoke VNet with subnets for private endpoints, application integration, agent subnet delegation, and management; private endpoints for every PaaS service; private DNS zones linked from the hub; egress through Azure Firewall |
| **Access** | Azure Bastion and a jump box, or a cloud PC (Windows 365, Azure Virtual Desktop) or Entra Private Access, for operators and developers |
| **Observability** | Application Insights and Log Analytics; Foundry tracing and evaluations |
| **Build** | Container Registry; build agents with network line of sight (for example Managed DevOps Pools or self-hosted runners in the VNet) |

Topic 34 walks through the baseline Foundry chat reference architecture component by component.

### Networking essentials

Networking is where AI landing zones most often go wrong.

**Private endpoints and DNS.** Every PaaS service is reached privately, and each needs name resolution through its private DNS zone:

| Service | Private DNS zone(s) |
|---|---|
| Foundry / Azure OpenAI / Foundry Tools | `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com` |
| Azure AI Search | `privatelink.search.windows.net` |
| Storage (blob) | `privatelink.blob.core.windows.net` |
| Cosmos DB | `privatelink.documents.azure.com` |
| Key Vault | `privatelink.vaultcore.azure.net` |
| Container Registry | `privatelink.azurecr.io` |
| API Management | `privatelink.azure-api.net` |

In a platform landing zone these zones live centrally in the connectivity subscription, and a policy (`DeployIfNotExists`) automatically registers each new private endpoint. Workload teams should not create their own copies of the zones.

**Public network access** is disabled on every service in *Corp*.

**Agent networking.** Foundry Agent Service in a network-isolated setup injects agent compute into a delegated subnet in your VNet, so agents reach your private data sources and their outbound traffic follows your routing. Tools that reach outside (web search, external MCP servers, OpenAPI tools) need deliberate egress rules.

**Egress control.** Route outbound traffic through Azure Firewall and allow only what is needed. Uncontrolled egress from an agent that can read internal data is an exfiltration path.

**Service-to-service trust.** Azure AI Search reaching Storage or Foundry privately uses shared private links or trusted-service exceptions. Plan these; they are a common source of "works in dev, fails in the isolated environment".

**On-premises data.** Use ExpressRoute for large or latency-sensitive transfers and VPN for moderate needs.

### Identity essentials

| Principle | Implementation |
|---|---|
| Keyless everywhere | Disable local authentication on Foundry, Search, Storage and Cosmos DB; use managed identities |
| Least privilege by role | Data-plane roles at resource scope (see topic 02, file 03) |
| Per-user data access | On-behalf-of flow and security trimming in retrieval |
| Agent identity | Each agent has its own Entra agent identity with narrow access and a sponsor |
| Developer access | Group-based, to non-production projects; production through pipelines and PIM |
| Separation of duties | Platform team manages policy and network; workload team manages its resources |

### Policy guardrails for AI

A practical baseline. Built-in definitions exist for most of these; exact names change, so search the built-in list by service.

| Guardrail | Effect | Applies to |
|---|---|---|
| Disable local (key) authentication | Deny | Foundry and Foundry Tools, Azure AI Search, Storage, Cosmos DB |
| Disable public network access | Deny (Corp) / Audit (Online) | Foundry, Search, Storage, Cosmos DB, Key Vault, Container Registry |
| Require private endpoints | Audit / DeployIfNotExists for DNS | All of the above |
| **Restrict which models can be deployed** | Deny | Foundry model deployments: allow an approved list of models and versions |
| Restrict deployment types | Deny | For example block Global deployments where in-region processing is required |
| Allowed regions | Deny | All resources |
| Require customer-managed keys | Audit or Deny | Where the data classification demands it |
| Send diagnostics to the central workspace | DeployIfNotExists | Foundry, Search, API Management, Key Vault |
| Enable Defender for Cloud AI protection | DeployIfNotExists | Subscriptions hosting AI |
| Restrict GPU VM SKUs | Deny | Sandbox and general workloads |
| Require tags | Deny or Modify | `owner`, `costCentre`, `dataClassification`, `application` |
| Restrict outbound access for Foundry Tools | Audit or Deny | Data-loss prevention on the resource |

The **model allow-list** is the AI-specific control clients ask about most. It lets a governance board approve models once (for licence terms, data handling, safety and cost) and have that decision enforced everywhere.

File 05 shows how these are written and assigned.

### Reliability and capacity

| Concern | Guidance |
|---|---|
| **Regions** | Choose for model availability and quota first; verify before finalising the design |
| **Multi-region** | Production AI endpoints commonly span at least two regions where criticality justifies it, load-balanced by API Management |
| **Quota** | Per subscription per region. Request increases early; consider dedicated subscriptions for large consumers |
| **Data placement** | Keep retrieval data in the same region as the model and application where possible |
| **Asset replication** | Replicate fine-tuned models, indexes, source data and agent definitions (as code) to the secondary region |
| **Zone redundancy** | Use zone-redundant tiers for Search, Cosmos DB, Storage and application hosting |

### Cost governance

- **Subscription or resource-group boundaries** for attribution, backed by tags.
- **Budgets and alerts** on every AI subscription from day one.
- **Token limits per consumer** at the gateway, so one application cannot consume the organisation's quota or budget.
- **Chargeback** from gateway metrics: tokens by application, team and model.
- **Provisioned throughput** purchased centrally where demand is steady, shared through the gateway.
- **Sandbox limits:** restrict expensive models and SKUs and cap spend.

### Sandbox for AI

People need somewhere to experiment that does not require a design review.

| Property | Sandbox setting |
|---|---|
| Network | No connection to the corporate network |
| Data | No production or sensitive data; synthetic or public only, enforced by policy and process |
| Models | A permitted list, with low quota |
| Budget | Hard-capped, with automatic alerts and expiry |
| Access | The requesting individual or team |
| Path to production | Rebuild in a governed workload landing zone from IaC; nothing is "promoted" from sandbox |

## How it works on Azure

### With and without a platform landing zone

| Situation | Approach |
|---|---|
| **Platform landing zone exists** | Vend a workload subscription under *Corp* or *Online*; deploy the AI workload into the provided spoke; rely on central DNS, firewall and logging; add AI policies at the management group |
| **No platform landing zone** | Use the standalone variant: the workload brings its own VNet, private DNS zones, firewall or NAT, Bastion and Log Analytics. Design it so it can be attached to a platform landing zone later |

Starting standalone is legitimate for a first production workload. Record the decisions that will change when a platform arrives (DNS zone ownership, egress path, logging destination).

### A day in the life of a new AI workload

1. A team submits a subscription request: owner, cost centre, exposure (internal or internet-facing), data classification, regions, expected models and volume.
2. Vending creates production and non-production subscriptions under the right management group, with spoke networks, RBAC, budgets and tags.
3. The team deploys the AI landing zone accelerator or its own IaC into the subscription.
4. Policies deny anything non-compliant at deployment time: public endpoints, keys, unapproved models.
5. Private endpoints register in central DNS automatically.
6. The team requests model quota, or consumes shared capacity through the gateway.
7. Agents are registered with identities and sponsors; logs flow to the central workspace; Defender for Cloud and Purview policies apply.
8. Before go-live, a review checks the design against the Well-Architected AI workload guidance.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Central gateway or direct access | Central gateway once more than a few teams consume models, or when chargeback, shared PTU or uniform safety policy is needed |
| Per-workload vs shared Foundry resource | Per-workload for production isolation; shared for a single team's related use cases |
| Global vs Data Zone vs Regional deployments | Decide from the residency requirement, then enforce with policy |
| Fully private vs public with controls | Fully private for internal data. For low-sensitivity internet-facing workloads, public endpoints with WAF, identity and firewall rules can be acceptable; make it an explicit decision |
| Who approves models | A lightweight AI governance board that maintains the allow-list |
| How strict in non-production | Same policies as production, so nothing is discovered at release time. Relax only cost-related SKUs |
| Standalone first or platform first | If AI is the forcing function and no platform exists, build a minimal platform landing zone at the same time; it is rarely wasted |

**The balance to strike:** governance heavy enough that security approves production, light enough that teams do not go around it. A sandbox with a clear, fast path into a governed landing zone achieves both.

## Hands-on

**Design exercise** (the most valuable one in this topic). For an organisation with an internal HR assistant and a public customer support agent:

1. Place both in the management group hierarchy and list their subscriptions.
2. Decide the Foundry topology: resources, projects, and whether a central gateway is used. Justify it.
3. Draw the spoke network for the HR assistant: subnets, private endpoints, the DNS zones needed, and the egress path.
4. Write the identity map: every identity, the role it holds, and the scope.
5. List ten policies with effect and scope.
6. State the regions, deployment types and what happens if the primary region fails.
7. Describe how costs are attributed and capped.

**Exploration**

```bash
# Built-in policies relevant to AI services
az policy definition list --query "[?policyType=='BuiltIn' && (contains(displayName,'Cognitive Services') || contains(displayName,'Azure AI') || contains(displayName,'Foundry'))].{Name:displayName, Effect:parameters.effect.defaultValue}" -o table

az policy definition list --query "[?policyType=='BuiltIn' && contains(displayName,'Search')].displayName" -o tsv

# Model availability and your quota in a candidate region
az cognitiveservices model list --location swedencentral \
  --query "[?kind=='OpenAI'].{Model:model.name, Version:model.version}" -o table | sort -u | head -40
az cognitiveservices usage list --location swedencentral \
  --query "[?limit > \`0\`].{Quota:name.value, Used:currentValue, Limit:limit}" -o table

# Audit existing AI resources for the two most important settings
az graph query -q "
Resources
| where type =~ 'microsoft.cognitiveservices/accounts'
| project name, kind, location, resourceGroup,
          publicAccess = tostring(properties.publicNetworkAccess),
          localAuthDisabled = tostring(properties.disableLocalAuth)" -o table
```

**Deployment (sandbox subscription):** find the current AI landing zone accelerator from the Cloud Adoption Framework *AI Ready* page or the Azure Architecture Center landing zone list, deploy the standalone variant, and inspect: the VNet and subnets, each private endpoint and its DNS record, the public network access setting on every service, and the role assignments between managed identities. Delete it afterwards; the firewall, gateway and search service have meaningful hourly cost.

## Security, cost and reliability notes

- **Security:** the three settings that matter most are public network access off, local authentication off, and controlled egress. Enforce all three with policy.
- **Security:** turn on Defender for Cloud's AI threat protection and connect Purview for data security in AI interactions (topic 23).
- **Security:** log prompts and responses only with a decision on retention, access and privacy. Those logs contain whatever users typed.
- **Cost:** the fixed cost of an isolated landing zone (firewall, Application Gateway, API Management, search, Bastion) is significant before a single token is processed. Share platform components across workloads where sensible.
- **Reliability:** DNS is the commonest failure cause in private AI environments. Test name resolution from every subnet and from on-premises.
- **Reliability:** a central gateway must be zone-redundant and, for critical workloads, multi-region.

## Common mistakes

- Creating a special management group or architecture for AI that ignores exposure and data sensitivity.
- Putting AI resources in platform subscriptions.
- Proof of concept promoted to production with keys and public endpoints intact.
- Each team creating its own private DNS zones, breaking resolution for everyone.
- Forgetting the service-to-service private paths (search to storage, search to model).
- Unrestricted agent egress.
- Choosing a region before checking model availability and quota.
- No model allow-list, so teams deploy whatever appears in the catalogue.
- One shared Foundry resource for production and development.
- Governance so slow that teams build outside it.

## Check yourself

1. Is an AI landing zone a different architecture from an Azure landing zone? Explain.
2. Under which management groups do internal and customer-facing AI workloads go, and why separate them?
3. Compare per-workload Foundry resources with a central gateway model.
4. Name four private DNS zones an isolated RAG workload needs.
5. What are the three most important security settings to enforce by policy?
6. What is a model allow-list and how is it enforced?
7. Why might an AI workload need its own subscription?
8. What properties should an AI sandbox have?

<details>
<summary>Answers</summary>

1. No. It is a workload landing zone configured for AI, with AI-specific policy added by the platform. The underlying landing zone architecture does not change.
2. Internal under Corp, customer-facing under Online. The separation creates a data governance boundary and allows different policies, for example denying public access for internal workloads.
3. Per-workload gives clear isolation, ownership and quota but duplicates setup and fragments capacity. A central gateway gives shared capacity, uniform policy and chargeback but adds a critical shared dependency.
4. Any four of the zones for Cognitive Services / OpenAI / AI services, Search, Blob storage, Cosmos DB, Key Vault, Container Registry.
5. Disable public network access, disable local (key) authentication, and control egress.
6. A list of approved models and versions, enforced with an Azure Policy deny on model deployments that are not on the list.
7. For dedicated model quota, cost attribution, blast-radius containment and a distinct compliance or network boundary.
8. No corporate network connectivity, no sensitive data, a limited model list and quota, a capped budget with expiry, and no promotion path other than rebuilding in a governed landing zone.

</details>

## References

- [AI Ready (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai/ready)
- [Govern AI](https://learn.microsoft.com/azure/cloud-adoption-framework/ai/govern)
- [Secure AI](https://learn.microsoft.com/azure/cloud-adoption-framework/ai/secure)
- [What is an Azure landing zone? (AI in Azure landing zones)](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [Baseline Microsoft Foundry chat reference architecture in an Azure landing zone](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone)
- [Control model deployments with Azure Policy](https://learn.microsoft.com/azure/foundry/how-to/model-deployment-policy)
- [Configure private link for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Set up private networking for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [AI gateway capabilities in Azure API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [Private Link and DNS integration at scale](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/private-link-and-dns-integration-at-scale)
