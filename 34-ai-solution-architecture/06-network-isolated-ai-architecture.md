# Network-Isolated AI Architecture

## Why it matters

Regulated organisations (banks, governments, healthcare, defence suppliers) usually require that sensitive AI workloads have **no public endpoints**, that data travels only over private networks, and that all outbound traffic is inspected and allow-listed. Achieving this for an AI solution is harder than for a web app: Foundry resources and projects, model endpoints, Agent Service compute and its dependencies (Cosmos DB, Storage, AI Search), knowledge sources, tools and MCP servers, evaluation and monitoring all need private connectivity and DNS that works, while developers still need to build and deploy. This file assembles a fully private Foundry, search, storage and agent architecture inside an Azure landing zone, building on [Topic 14 file 04](../14-microsoft-foundry-platform/04-foundry-networking.md) and [Topic 05 file 18](../05-networking/18-networking-for-ai-workloads.md).

## Key concepts

### Isolation layers

| Layer | Control |
|---|---|
| **Inbound to Foundry** | Private endpoint for the Foundry resource (account + projects); public network access disabled |
| **Inbound to dependencies** | Private endpoints for Cosmos DB, Storage (blob/file), AI Search, Key Vault, ACR, App Configuration, Application Insights (Azure Monitor Private Link Scope) |
| **Agent compute outbound** | **Network injection (BYO VNet)**: agent compute in your delegated subnet (`Microsoft.App/environments`) with NSGs and UDRs to Azure Firewall; or **managed virtual network** with managed private endpoints and isolation mode |
| **Application tier** | App Service/Container Apps with VNet integration and private ingress; Application Gateway WAF as the only public entry (or none for internal apps) |
| **Egress** | Azure Firewall with FQDN allow-lists for external tools/MCP servers; no direct internet from AI subnets |
| **DNS** | Private DNS zones (for example `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com`, `privatelink.search.windows.net`, `privatelink.documents.azure.com`, `privatelink.blob.core.windows.net`) linked centrally; DNS Private Resolver for on-premises |
| **Management access** | Bastion/jump hosts or self-hosted build agents in the VNet for deployments; Entra-only auth (local auth disabled) |
| **Policy** | Azure Policy denying public network access, requiring private endpoints, disabling local auth |

### Standard (network-isolated) agent setup

The standard agent setup uses **your** Cosmos DB, Storage and AI Search resources (all private) so conversation data, files and vector stores stay in your network boundary. The agent subnet must be able to reach them via private endpoints and resolve their private DNS names.

### Landing zone responsibilities

| Platform team (connectivity subscription) | Workload team (application landing zone) |
|---|---|
| Hub VNet, Azure Firewall rules, DNS zones and resolver, ExpressRoute/VPN | Spoke VNet, subnets (app, private endpoints, agent delegation), NSGs |
| Policies (deny public access, require PE) | Foundry resource/projects, models, AI Search, Cosmos DB, Storage, Key Vault |
| Central monitoring, Defender | Application, agents, evaluation, workload monitoring |
| Firewall rule requests process | Requests FQDNs for tools and dependencies |

### Developer experience in isolated environments

Developers need access: VPN/ExpressRoute or Entra Private Access to reach the Foundry portal's private endpoints, self-hosted CI agents (Managed DevOps Pools or GitHub runners in the VNet) to deploy agents and run evaluations, and sometimes a less restricted dev environment with synthetic data.

## How it works on Azure

Reference topology:

```text
Hub (platform): Azure Firewall Premium, DNS Private Resolver, private DNS zones, ExpressRoute
Spoke (workload):
  snet-appgw   : Application Gateway WAF (public or private frontend)
  snet-app     : App Service VNet integration (chat UI/API)
  snet-agents  : delegated to Microsoft.App/environments (Foundry Agent Service network injection), UDR 0.0.0.0/0 → firewall
  snet-pe      : private endpoints → Foundry, AI Search, Cosmos DB, Storage, Key Vault, ACR, AMPLS
  snet-build   : self-hosted runners / Managed DevOps Pool
Egress allow-list: approved MCP servers/APIs only; model calls stay private via Foundry private endpoint
Policy: deny publicNetworkAccess on Cognitive Services, Search, Cosmos DB, Storage; audit local auth
```

Policy example (built-in): "Azure AI Services resources should restrict network access" and "Azure AI Services resources should have key access disabled (disable local authentication)" assigned with Deny at the landing zone management group (verify names and IDs in your tenant).

Validation checklist:

- `nslookup <foundry>.services.ai.azure.com` from the app subnet resolves to a private IP.
- Agent tool calls to AI Search succeed; calls to unapproved internet FQDNs are blocked and logged by the firewall.
- Public network access returns 403 from the internet.
- CI pipeline deploys agents from the in-VNet runner.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **BYO VNet vs managed VNet** | BYO VNet for full control and integration with hub firewall/DNS; managed VNet for simplicity where supported. |
| **Internal vs internet-facing UI** | Internal apps behind private Application Gateway or Entra Private Access; external apps with WAF. |
| **Tool egress** | Prefer private MCP servers/APIs; allow-list external FQDNs explicitly. |
| **Feature availability** | Some Foundry features (certain tools, Preview capabilities) may not support full network isolation; check before committing. |
| **Cost** | Firewall, private endpoints, DNS resolver and Premium SKUs add cost; share via hub. |
| **Developer productivity** | Provide secure connectivity and in-VNet tooling; don't force public exceptions. |

## Hands-on

1. Deploy a Foundry resource with public access disabled and a private endpoint; verify DNS resolution.
2. Configure the standard agent setup with private Cosmos DB, Storage and AI Search and network injection.
3. Route agent subnet egress through Azure Firewall; allow one MCP server FQDN; test blocked and allowed calls.
4. Assign deny policies for public network access and local auth.
5. Run an evaluation pipeline from a self-hosted runner inside the VNet.

## Security, cost and reliability notes

- **DNS is the most common failure**: centralise zones and test resolution from every subnet.
- **Exfiltration risk**: egress allow-lists and Prompt Shields for indirect injection.
- **Reliability**: private endpoints and firewalls must be zone-redundant; plan multi-region DNS.
- **Cost**: budget for networking components; monitor firewall throughput charges.

## Common mistakes

| Mistake | Fix |
|---|---|
| Private Foundry but public dependencies | Private endpoints for all dependencies |
| Missing private DNS zones | Central zones linked to VNets |
| Direct internet egress from agent subnet | UDR to firewall |
| No way for developers to reach private portal | VPN/Private Access and in-VNet runners |
| Assuming every feature supports isolation | Check feature support |

## Check yourself

1. List the isolation layers for an AI workload.
2. What is network injection for Foundry agents?
3. Which dependencies does the standard agent setup require privately?
4. Which private DNS zones are typically needed?
5. How are responsibilities split between platform and workload teams?
6. How do developers work in a fully private environment?

<details>
<summary>Answers</summary>

1. Inbound to Foundry, inbound to dependencies, agent outbound, application tier, egress, DNS, management access, policy.
2. Injecting Agent Service compute into your delegated subnet so it uses your network, NSGs, UDRs and DNS.
3. Cosmos DB, Storage and AI Search (plus Key Vault and others as used).
4. Cognitive Services/OpenAI/services.ai zones, Search, Cosmos DB documents, Blob/File, Key Vault, ACR, Azure Monitor zones.
5. Platform: hub, firewall, DNS, connectivity, policy, central monitoring; workload: spoke, Foundry and data resources, app, agents, evaluation.
6. Via VPN/ExpressRoute or Entra Private Access to private endpoints and in-VNet CI runners, sometimes a dev environment with synthetic data.

</details>

## References

- [Baseline Foundry chat in an Azure landing zone](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone)
- [Configure private link for Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Virtual networks for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [Managed virtual network for Foundry](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [Secure networking for Azure AI platforms (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/platform/networking)
