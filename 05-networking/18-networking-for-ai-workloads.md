# Networking for AI Workloads

## Why it matters

This file applies the whole networking topic to an AI application. A production RAG or agent system that handles internal data has to answer four questions to pass a security review: how do users get in, how do the components reach each other privately, what can the system reach on the internet, and how do people and pipelines work inside the boundary. Foundry networking is evolving, so treat the specifics as a snapshot and confirm against current documentation before finalising a design.

## Key concepts

### The four traffic planes

| Plane | Traffic | Controlled by |
|---|---|---|
| **1. Ingress** | Users → application | Front Door or Application Gateway with WAF; authentication |
| **2. Application → AI and data** | App and agents → Foundry, Search, Storage, Cosmos DB, Key Vault | Private endpoints, private DNS, NSGs, managed identities |
| **3. Service → service** | Search → Storage and Foundry; Foundry → Search, Storage, Cosmos DB | Shared private links, trusted-service access, network security perimeter, agent VNet injection |
| **4. Egress** | App and agents → internet; agent tools → external APIs | Private subnets, NAT gateway or firewall with FQDN allow-list |

Plus a fifth concern: **access for people and pipelines** (developers, operators, CI/CD).

Most first attempts get plane 2 right and miss plane 3. That produces the familiar result: the application can call the model, but indexers cannot read documents and agents cannot reach their knowledge.

### Isolation levels

Choose deliberately; do not default to the most complex.

| Level | Description | Fits |
|---|---|---|
| **Public with controls** | Public endpoints, Entra authentication, keys disabled, IP allow-lists where practical | Prototypes, public data, early development |
| **Private inbound** | Private endpoints on all PaaS resources, public access disabled; outbound not tightly controlled | Internal data where exfiltration risk is moderate |
| **Fully isolated** | Private inbound **and** controlled outbound for applications, agents and PaaS services; no public entry except a WAF | Regulated or sensitive data; most enterprise production |

Each level up adds cost, deployment time and operational friction. The data classification and the organisation's policy decide.

### Private access to Foundry

A Foundry resource (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) and its projects are made private the standard way:

1. A **private endpoint** with sub-resource `account` in the workload's private endpoint subnet.
2. Records in **three private DNS zones**:
   - `privatelink.cognitiveservices.azure.com`
   - `privatelink.openai.azure.com`
   - `privatelink.services.ai.azure.com`
3. `publicNetworkAccess: Disabled`.
4. `disableLocalAuth: true`, so only Entra tokens work.

This covers **inbound** access to the resource: model inference, the project API, and the data-plane calls the Foundry portal makes from your browser. With public access disabled, the portal only works from a machine with private line of sight.

Two properties worth knowing for outbound control of the resource itself:

- **`restrictOutboundNetworkAccess`** with **`allowedFqdnList`** limits where the service may call out, useful for features that fetch from URLs.
- **Network security perimeter** association, where supported, for central inbound and outbound rules (file 16).

### Deployment type and the network

A private endpoint controls how **you reach** the resource. It does not change **where inference runs**. With a Global or Data Zone deployment, your request enters privately and is then processed on Microsoft's network in whichever region has capacity within that scope. That is a data-processing-location question (topic 03, file 08), not a network-exposure one. Do not conflate "private endpoint" with "stays in region".

### Networking for Foundry Agent Service

Agents add a component that makes outbound calls on your behalf: to your knowledge sources, to tools, to MCP servers. Where that runtime sits on the network is the key design decision.

| Setup | Agent runtime | State and knowledge resources | Use |
|---|---|---|---|
| **Basic** | Microsoft-managed, multi-tenant | Microsoft-managed | Getting started |
| **Standard** | Microsoft-managed | **Your own** resources: Storage (files), Azure AI Search (vector stores), Cosmos DB (threads) | Data control without full isolation |
| **Standard with private networking** (bring your own virtual network) | Injected into a **delegated subnet in your VNet** | Your own resources, reached by **private endpoints** | Network-isolated production |

In the private networking setup:

- You provide a **subnet delegated** to the agent service's resource provider (`Microsoft.App/environments`), sized generously (a `/24` is commonly recommended), in the same region as the Foundry resource.
- Agent compute is **injected** into that subnet, so its outbound traffic originates from your VNet and follows your **route tables, NSGs and firewall**.
- The agent reaches your Storage, Search and Cosmos DB through their **private endpoints**.
- Your application reaches the agent through the Foundry **private endpoint**.

Consequences to plan for:

| Topic | Implication |
|---|---|
| **Egress** | Tools that call out (OpenAPI tools, remote MCP servers, web grounding) leave through your egress path. Allow-list their FQDNs at the firewall |
| **Tool support** | Not every built-in tool works identically in a network-isolated setup, and some rely on Microsoft-hosted services. Check the current support matrix for each tool you intend to use |
| **Private tools** | MCP servers and APIs you host on Functions, Container Apps or API Management should be private too, reachable from the agent subnet |
| **Region and capacity** | The subnet, Foundry resource and dependencies have regional constraints |
| **Subnet exclusivity** | The delegated subnet is for the agent service alone |
| **Hosted agents** | Agents you deploy as your own code run in managed sandboxes; verify their networking options and whether they meet your isolation requirement |

Microsoft also offers a **managed virtual network** option for Foundry in which the platform manages the network and you define outbound rules, as an alternative to bringing your own VNet. It trades control for simplicity. Check its current status and limits.

### Service-to-service paths

The PaaS services call each other from Microsoft-managed infrastructure, not from your VNet. Each path needs a mechanism:

| Caller → target | Purpose | Options |
|---|---|---|
| **AI Search → Storage / Cosmos DB / SQL** | Indexers read source data | Shared private link (search creates a private endpoint to the target, you approve); trusted-service or resource-instance rule on storage; network security perimeter |
| **AI Search → Foundry** | Integrated vectorisation, skills calling models | Shared private link; trusted-service configuration; perimeter |
| **Foundry (agent or model features) → AI Search** | Retrieval, knowledge bases, "on your data" | Agent VNet injection with private endpoint; trusted-service access on search; perimeter |
| **Foundry → Storage** | Files, batch input and output, fine-tuning data | Agent VNet injection; trusted-service or resource-instance rule; perimeter |
| **API Management → Foundry** | Gateway to models | APIM in the VNet (injection or integration) with a private endpoint to Foundry |
| **Any → Key Vault** | Customer-managed keys, secrets | Private endpoint from the VNet; trusted services for platform CMK access |
| **Any → Log Analytics / Application Insights** | Telemetry | Public ingestion, or Azure Monitor Private Link Scope |

All of these also need **RBAC**: the calling service's managed identity must hold the data role on the target (topic 02, file 03). A correct network path with a missing role fails just as completely, with a different error.

Two notes on Azure AI Search:

- **Shared private links** have tier requirements (Basic and above, with some targets or skillset scenarios needing Standard 2 or higher), and indexers with skillsets may need to run in the **private execution environment**. Confirm for your tier.
- A **trusted-service** exception is simpler than a shared private link where the target supports it and policy allows.

### Application hosting on the network

| Host | Inbound | Outbound to private resources |
|---|---|---|
| **Container Apps** | Internal environment in your VNet, fronted by Application Gateway; or a private endpoint on the environment | The environment's subnet; UDR to firewall supported on workload profiles environments |
| **App Service / Functions** | Private endpoint, public access off | **VNet integration** with *route all* enabled |
| **AKS** | Internal load balancer or Application Gateway for Containers | Node and pod subnets; egress via `userDefinedRouting` or NAT gateway |
| **API Management** | VNet injection (classic Premium, Premium v2) or private endpoint; internal mode behind Application Gateway | Injection or outbound VNet integration (Standard v2, Premium v2) |

Whatever the host, set its DNS so private zones resolve, and confirm with `nslookup` from inside the running app.

### Ingress and streaming

Chat applications stream tokens with **server-sent events**, and voice applications use **WebSockets** or **WebRTC**.

| Concern | Guidance |
|---|---|
| **Buffering** | Disable response buffering on Application Gateway, API Management and any reverse proxy for streaming routes, or users see the whole answer arrive at once |
| **Timeouts** | Raise backend, idle and origin response timeouts. Long generations and agent runs exceed defaults. Check every hop; the shortest wins |
| **WebSockets** | Supported by Application Gateway, Front Door and Container Apps; confirm on each hop and that idle timeouts allow long sessions |
| **WebRTC** (real-time audio) | Media flows over UDP and does not traverse HTTP proxies. The client talks to the service's media endpoints; only signalling goes through your app. Plan network egress on client networks accordingly |
| **WAF** | Add exclusions for prompt bodies on the chat route; raise body size limits; rate-limit per client |
| **Large uploads** | Send files directly to Blob Storage with user delegation SAS or through a dedicated upload path, not through the chat API |
| **API Management policies** | Policies that read the response body force buffering. For streaming APIs, avoid them on the response path and use the streaming-aware token metric and limit policies |

### Egress for agents

Agents make egress control a first-order requirement. An agent that can read internal documents and can also make arbitrary outbound web requests is an exfiltration channel waiting for a prompt injection.

- **Default deny.** Private subnets; all outbound through the firewall.
- **Allow-list by FQDN** exactly the tools, MCP servers and APIs the agent is designed to use.
- **Prefer private tools.** Host your own MCP servers and APIs inside the network.
- **Treat web browsing and search tools as high risk** for agents with access to sensitive data. Either separate them into a different agent without that access, or accept and document the risk.
- **Log** every outbound connection and review.
- Network controls bound the damage; they do not replace least-privilege identities and human approval for consequential actions (topic 23).

### Latency

Network placement affects response time directly.

| Hop | Guidance |
|---|---|
| App ↔ model | Same region where possible. Private endpoints add negligible latency |
| App ↔ search ↔ data | Co-locate. Retrieval makes several round trips per request |
| Cross-region model access | Common when a model is unavailable locally. Traffic stays on the Microsoft backbone; add tens of milliseconds, which is small against generation time |
| Via the hub firewall | Adds a little latency and cost. Private endpoint traffic within the spoke normally bypasses it |
| On-premises data in the request path | Often the slowest hop. Replicate or cache |
| API Management in the path | Small overhead; worthwhile for the control it gives |

Time to first token matters more to users than total time. Stream early and keep the retrieval path short.

## How it works on Azure

### Reference layout: fully isolated RAG and agent workload

```text
                         Internet
                            │
                  Front Door Premium (WAF)                      [optional, global]
                            │ Private Link
┌──────────────────────── Spoke VNet 10.20.0.0/22 ───────────────────────────────┐
│ snet-appgw      Application Gateway WAF_v2 (private or public frontend)        │
│ snet-apps       Container Apps (internal) ─ app API, orchestration             │
│ snet-agents     Foundry Agent Service (delegated, injected)                    │
│ snet-apim       API Management (AI gateway)                     [optional]     │
│ snet-pe         Private endpoints:                                             │
│                   Foundry (account) · AI Search · Storage (blob) · Cosmos DB   │
│                   Key Vault · Container Registry                               │
│ snet-mgmt       Jump box / VNet-connected build agents                         │
│ NSG per subnet · route table: 0.0.0.0/0 → hub firewall                         │
└───────────────────────────────┬────────────────────────────────────────────────┘
                        peering │
┌───────────────────────── Hub VNet ─────────────────────────────────────────────┐
│ Azure Firewall (FQDN allow-list, DNS proxy) · DNS Private Resolver · Bastion   │
│ ExpressRoute / VPN gateway · private DNS zones (linked here)                   │
└─────────────────────────────────────────────────────────────────────────────────┘
        Network security perimeter around the PaaS resources (where supported)
```

### The checklist

**Private endpoints and DNS zones**

| Resource | Sub-resource | Zone(s) |
|---|---|---|
| Foundry | `account` | `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com` |
| Azure AI Search | `searchService` | `privatelink.search.windows.net` |
| Storage | `blob` (and `file`, `dfs` if used) | `privatelink.blob.core.windows.net` (and equivalents) |
| Cosmos DB | `Sql` | `privatelink.documents.azure.com` |
| Key Vault | `vault` | `privatelink.vaultcore.azure.net` |
| Container Registry | `registry` | `privatelink.azurecr.io` |
| Container Apps environment (if private endpoint used) | `managedEnvironments` | Region-specific `privatelink` zone |
| API Management | `Gateway` | `privatelink.azure-api.net` |
| Azure Monitor (optional) | via AMPLS | Five Azure Monitor zones |

**Settings on every PaaS resource**

- [ ] `publicNetworkAccess` disabled (or secured by perimeter)
- [ ] Local or key authentication disabled
- [ ] Managed identity enabled
- [ ] Diagnostic settings to the central workspace

**Service-to-service**

- [ ] Search → Storage: path and role (*Storage Blob Data Reader*)
- [ ] Search → Foundry: path and role (*Cognitive Services OpenAI User*)
- [ ] Foundry project / agent → Search: path and role (*Search Index Data Reader* or *Contributor*)
- [ ] Foundry project / agent → Storage and Cosmos DB: path and role
- [ ] App → Foundry, Search, Cosmos DB, Key Vault: private endpoint and role
- [ ] App host → Container Registry: private endpoint and *AcrPull*

**Egress**

- [ ] Private subnets everywhere
- [ ] Route table to the firewall on app, agent and management subnets
- [ ] FQDN allow-list written and reviewed, including each agent tool
- [ ] Firewall logs flowing

**Ingress**

- [ ] WAF in Prevention with tuned exclusions
- [ ] Origin not reachable except through the entry point
- [ ] Buffering off and timeouts raised for streaming routes
- [ ] Rate limits and authentication before model calls

**People and pipelines**

- [ ] Developer access path, with DNS
- [ ] Build agents with network line of sight
- [ ] Bastion or equivalent for operators

**Verification**

- [ ] From the app: every dependency resolves to a private IP
- [ ] From the internet: every PaaS endpoint refuses
- [ ] An indexer run succeeds
- [ ] An agent run using knowledge and each tool succeeds
- [ ] A denied egress attempt appears in the firewall log

Run the verification list in a non-production environment that has the **same policies and network design** as production. The gap between a permissive dev environment and an isolated production one is where go-live surprises come from.

### Shared and multi-workload patterns

| Pattern | Network implications |
|---|---|
| **Central AI gateway** | API Management in a shared AI platform spoke, with private endpoints to Foundry resources (possibly in several regions). Workload spokes reach the gateway through the hub or a private endpoint |
| **Shared model capacity across regions** | Private endpoints to each regional Foundry resource from the gateway's VNet; cross-region private endpoints are supported |
| **Per-workload Foundry** | Each spoke has its own private endpoints and DNS records in the central zones |
| **Hybrid consumers** | On-premises applications call the gateway over ExpressRoute; DNS forwarders resolve the gateway and Foundry names |

Topic 34 develops these architectures.

### Cost of isolation

A realistic fixed monthly footprint before any tokens are used: Application Gateway WAF_v2, Azure Firewall (shared in the hub), eight or so private endpoints, a NAT gateway or firewall data processing, Bastion (shared), DNS Private Resolver endpoints (shared), a jump box or managed workstations, and API Management if used. Sharing hub components across workloads is what makes this affordable. For a small pilot, the network can cost more than the AI. Say so in the estimate.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Isolation level | From data classification and policy. Fully isolated for sensitive internal data |
| Bring your own VNet or managed network for agents | Bring your own for maximum control and integration with a landing zone. Managed for simplicity where its outbound controls suffice |
| Shared private link, trusted service, or perimeter | Trusted-service access is simplest. Shared private links where a fully private path is mandated. Perimeter where all resources and scenarios are supported |
| Inspect private endpoint traffic at the firewall | Usually no within a workload |
| Regional or global ingress | Application Gateway for internal and single-region. Front Door for global or edge protection |
| API Management in the path | Yes when several consumers share models, or you need token limits, load balancing and central logging |
| Model in another region | Acceptable for latency. Check data-processing requirements |
| Developer experience | Budget for it: a workstation inside the boundary, or a well-configured VPN with DNS. An isolated environment nobody can work in gets bypassed |
| Non-production parity | Same network design and policies as production, smaller SKUs |

**The honest trade-off.** Full isolation roughly doubles the infrastructure work of an AI project and lengthens every debugging session. It is the right call for sensitive data, and it is over-engineering for a public FAQ bot. Decide per workload, and use accelerators (the AI landing zone, the baseline reference implementation, Azure Verified Modules) so you are not hand-building it.

## Hands-on

**Design exercise (primary).** For the HR assistant used throughout this roadmap (internal users, HR policy documents in Blob Storage, an agent that can look up leave balances in an on-premises HR system through an API, one MCP server you host):

1. Choose the isolation level and justify it.
2. Draw the spoke: subnets with sizes and delegations.
3. List every private endpoint, its sub-resource and its DNS zones.
4. Fill in the service-to-service table: for each path, the mechanism and the role.
5. Write the firewall egress allow-list.
6. Describe ingress: entry point, WAF settings, streaming configuration.
7. Describe how a developer, an operator and the deployment pipeline each get in.
8. Write the ten-line verification script you would run from inside the app.

**Build exercise.** Deploy a current network-isolated reference implementation into a sandbox subscription: the *baseline Microsoft Foundry chat reference architecture* from the Azure Architecture Center, or the AI landing zone accelerator. Budget a few hours and delete it the same day.

Then verify from the jump box:

```bash
# 1. Every dependency resolves privately
for H in <foundry>.openai.azure.com <foundry>.cognitiveservices.azure.com <foundry>.services.ai.azure.com \
         <search>.search.windows.net <storage>.blob.core.windows.net \
         <cosmos>.documents.azure.com <kv>.vault.azure.net <acr>.azurecr.io; do
  printf "%-55s %s\n" "$H" "$(dig +short $H | tail -1)"
done

# 2. A model call over the private path, keyless
TOKEN=$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)
curl -s https://<foundry>.openai.azure.com/openai/v1/chat/completions \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"<deployment>","messages":[{"role":"user","content":"ping"}]}' | head -c 300

# 3. Streaming arrives incrementally through the ingress
curl -N -s https://<app-hostname>/api/chat -H "Content-Type: application/json" \
  -d '{"message":"Count slowly to ten."}'

# 4. Egress is denied by default
curl -s -m 8 https://example.org -o /dev/null -w "%{http_code}\n" || echo "blocked"
```

And from your own laptop, outside the network:

```bash
dig +short <foundry>.openai.azure.com          # public IP
curl -s -o /dev/null -w "%{http_code}\n" https://<foundry>.openai.azure.com/   # 403
```

Finally, run an indexer and an agent with a knowledge tool, and find the denied `example.org` attempt in the firewall logs.

**Audit exercise.** Run this against an existing subscription to find AI resources that are not isolated:

```bash
az graph query -q "
Resources
| where type in~ ('microsoft.cognitiveservices/accounts','microsoft.search/searchservices',
                  'microsoft.storage/storageaccounts','microsoft.documentdb/databaseaccounts','microsoft.keyvault/vaults')
| extend publicAccess = tostring(properties.publicNetworkAccess),
         privateEndpoints = array_length(properties.privateEndpointConnections),
         localAuthDisabled = tostring(properties.disableLocalAuth)
| project type, name, resourceGroup, publicAccess, privateEndpoints, localAuthDisabled
| order by type asc" -o table
```

## Security, cost and reliability notes

- **Security:** network isolation is one layer. Identity (keyless, least privilege, per-user data access), content safety and evaluation are the others. A private network with an over-privileged agent is still unsafe.
- **Security:** controlled egress is the control that most directly limits data exfiltration by a manipulated agent.
- **Security:** do not leave "temporary" public access exceptions in place. Enforce with policy.
- **Cost:** shared hub services are the main lever. Right-size non-production; stop what can be stopped.
- **Reliability:** DNS and service-to-service paths cause most outages in isolated AI environments. Monitor them explicitly.
- **Reliability:** zone-redundant ingress, firewall and private endpoints; plan the second region's network (its own spoke, endpoints and DNS records) if multi-region is required.

## Common mistakes

- Securing the application's path to the model and forgetting indexers and agents.
- Missing one of Foundry's three DNS zones.
- Equating a private endpoint with in-region processing.
- Unrestricted egress from agent subnets.
- Assuming every agent tool works in a network-isolated setup without checking.
- Buffering proxies and short timeouts in front of streaming endpoints.
- A private environment with no workable developer access, leading to public exceptions.
- Hosted build agents that cannot reach private resources.
- Correct network, missing RBAC role (or the reverse), and blaming the wrong layer.
- Development in a public environment and first contact with isolation at go-live.
- Hand-building all of this instead of starting from a reference implementation.
- Applying full isolation to a workload that did not need it.

## Check yourself

1. Name the four traffic planes of an AI workload.
2. What is needed to make a Foundry resource privately reachable?
3. Does a private endpoint determine where inference is processed? Explain.
4. In the network-isolated agent setup, where does agent compute run and why does that matter?
5. How can Azure AI Search reach a storage account that has public access disabled?
6. What must be configured on proxies in front of a streaming chat endpoint?
7. Why is egress control especially important for agents?
8. What should a verification pass confirm before go-live?
9. When is full network isolation not warranted?

<details>
<summary>Answers</summary>

1. Ingress (users to app), application to AI and data, service to service, and egress.
2. A private endpoint with the `account` sub-resource, records in three private DNS zones (`cognitiveservices`, `openai`, `services.ai`), public network access disabled, and local auth disabled.
3. No. It controls the network path to the resource. Where inference runs is set by the deployment type (Regional, Data Zone or Global).
4. In a subnet in your virtual network delegated to the agent service, so its outbound traffic follows your routes, NSGs and firewall and can reach your private endpoints.
5. Through a shared private link, a trusted-service or resource-instance rule on the storage account, or by placing both in a network security perimeter; plus an RBAC role for the search service's identity.
6. Response buffering disabled, timeouts raised on every hop, WAF exclusions and body limits adjusted, and WebSocket support confirmed where used.
7. An agent with access to internal data and unrestricted outbound access can be manipulated by prompt injection into sending that data out. An FQDN allow-list bounds where it can go.
8. That every dependency resolves to a private IP from inside, public endpoints refuse from outside, indexers and agent tools work end to end, streaming works through the ingress, and denied egress is logged.
9. For prototypes and workloads handling only public or low-sensitivity data, where the cost and friction outweigh the risk reduction.

</details>

## References

- [Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
- [Baseline Microsoft Foundry chat reference architecture in an Azure landing zone](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone)
- [Configure private link for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Set up private networking for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [Configure Azure AI services virtual networks](https://learn.microsoft.com/azure/ai-services/cognitive-services-virtual-networks)
- [Make outbound connections through a shared private link (Azure AI Search)](https://learn.microsoft.com/azure/search/search-indexer-howto-access-private)
- [Make indexer connections to Azure Storage as a trusted service](https://learn.microsoft.com/azure/search/search-indexer-howto-access-trusted-service-exception)
- [AI Ready: networking (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai/ready)
- [Networking in Azure Container Apps environment](https://learn.microsoft.com/azure/container-apps/networking)
- [Use a virtual network with Azure API Management](https://learn.microsoft.com/azure/api-management/virtual-network-concepts)
- [Azure Well-Architected Framework: AI workloads, networking](https://learn.microsoft.com/azure/well-architected/ai/networking)
