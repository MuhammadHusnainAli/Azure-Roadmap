# Foundry Networking

> Topic: 14 · Microsoft Foundry Platform · Tracks: E ○ A ● C ○ · Status checked: 2026-11

## Why it matters

Enterprises rarely allow AI platforms on the public internet. Prompts contain business data, agents call internal APIs, and regulators ask where traffic flows. Foundry networking has two halves. **Inbound**: who can reach the Foundry endpoints, which you control with private endpoints and public network access. **Outbound**: how agents, tools and evaluations reach your private resources and the internet, which you control with network injection or a managed virtual network and egress rules. Getting both right is a standard part of an AI landing zone (topic 03 file 04, topic 05 file 18).

## Key concepts

### Inbound: private endpoints

A **private endpoint** gives the Foundry resource a private IP in your VNet. With **public network access disabled**, the resource is reachable only through private endpoints (and trusted Azure services where allowed).

| Private DNS zone | Endpoint served |
|---|---|
| `privatelink.cognitiveservices.azure.com` | Foundry Tools / Cognitive Services endpoints |
| `privatelink.openai.azure.com` | OpenAI endpoint (`*.openai.azure.com`) |
| `privatelink.services.ai.azure.com` | Foundry project and AI services endpoint (`*.services.ai.azure.com`) |

One private endpoint (sub-resource `account`) on the Foundry resource covers its endpoints. The three DNS zones must resolve correctly from every client network (hub-and-spoke DNS, Private DNS Resolver for on-premises). Use the portal-generated DNS zone group or Azure Policy-driven DNS integration.

**Portal access:** when public access is disabled, the Foundry portal works only from a network that can reach the private endpoint (VPN, ExpressRoute, Bastion jump box, or Azure Virtual Desktop). Some network-isolated configurations must be set up through the SDK or CLI rather than the portal.

### Outbound: agent and tool traffic

Agent Service runs your agents' tools and code in Microsoft-managed container infrastructure. To keep that traffic private, Foundry supports two models:

| Model | How it works | Trade-off |
|---|---|---|
| **Customer-managed VNet (network injection, "BYO VNet")** | You provide a VNet and a dedicated subnet **delegated to `Microsoft.App/environments`**; the platform injects agent compute into it. Agents reach your private resources (AI Search, Storage, Cosmos DB, internal APIs, MCP servers) through your network and DNS | Full control (NSGs, UDRs to Azure Firewall, DNS); you manage the network |
| **Managed virtual network** | Foundry creates and manages the VNet; you approve **managed private endpoints** to your resources and choose an **isolation mode** for internet egress | Simpler; less customisation (check current status and supported scenarios) |

### Network-isolated (standard) agent setup

For a fully private agent deployment, Microsoft's **standard agent setup with private networking** uses your own resources for agent state:

| Component | Purpose | Network requirement |
|---|---|---|
| Foundry resource + project | Agent runtime and endpoints | Private endpoint; public access disabled |
| **Azure Storage** | Files uploaded to agents | Private endpoint (blob) |
| **Azure Cosmos DB for NoSQL** | Conversation (thread) storage | Private endpoint |
| **Azure AI Search** | Vector stores and knowledge | Private endpoint |
| Agent subnet | Injected agent compute | Delegated to `Microsoft.App/environments`, sized per guidance (for example /24) |
| Private endpoint subnet | Private endpoints for all of the above | Standard subnet |
| Private DNS zones | Name resolution for every service | Linked to the VNet |

Microsoft publishes Bicep and Terraform templates for this setup (the "standard agent setup with network isolation" samples). Region, subnet size and address ranges have constraints; follow the template's README.

### Egress control

| Control | Use |
|---|---|
| **Azure Firewall / NVA** with UDRs on the agent subnet | Allow-list FQDNs for external tools (web APIs, MCP servers) and log all egress |
| **NSGs** | Restrict traffic between subnets |
| **Managed VNet isolation mode** | "Allow only approved outbound" plus FQDN or private endpoint rules (managed VNet) |
| **Network Security Perimeter** | Perimeter-based access control for supported PaaS services (topic 05 file 16; check Foundry support) |
| **AI gateway (APIM)** | Route model and tool calls through a governed egress point (topic 11 file 07) |

Tools that call the public internet (for example web search grounding or public MCP servers) need explicit egress. Decide deliberately which tools are allowed in isolated environments.

### Trusted services and connected resources

- Connected resources (Storage, AI Search, Key Vault, Cosmos DB) have **their own** network settings. Lock them down independently.
- Some Azure services (for example AI Search indexers calling Foundry for embeddings) need **trusted service** exceptions or **shared private links** to reach a private Foundry resource.
- Use managed identities across all of these connections.

## How it works on Azure

### Private endpoint for a Foundry resource (Bicep sketch)

```bicep
resource foundry 'Microsoft.CognitiveServices/accounts@2025-06-01' existing = { name: foundryName }

resource pe 'Microsoft.Network/privateEndpoints@2024-05-01' = {
  name: 'pe-${foundryName}'
  location: location
  properties: {
    subnet: { id: peSubnetId }
    privateLinkServiceConnections: [ {
      name: 'foundry'
      properties: { privateLinkServiceId: foundry.id, groupIds: [ 'account' ] }
    } ]
  }
}

resource dnsGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2024-05-01' = {
  parent: pe
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      { name: 'cogsvc',   properties: { privateDnsZoneId: zoneCognitiveServicesId } }
      { name: 'openai',   properties: { privateDnsZoneId: zoneOpenAiId } }
      { name: 'aiservices', properties: { privateDnsZoneId: zoneServicesAiId } }
    ]
  }
}
```

Set `publicNetworkAccess: 'Disabled'` on the Foundry resource (and `networkAcls.defaultAction: 'Deny'`). For agent network injection, the account's `networkInjections` property references the delegated subnet. Use Microsoft's templates for the exact schema and API version.

### Reference topology

```text
On-prem ──ExpressRoute/VPN──► Hub VNet (Azure Firewall, DNS Private Resolver, Bastion)
                                   │ peering
                    ┌──────────────┴──────────────────────────────┐
                    │ AI spoke VNet                                │
                    │  snet-pe:     PE Foundry · PE Search · PE Storage · PE Cosmos DB · PE Key Vault │
                    │  snet-agents: delegated Microsoft.App/environments (agent injection)            │
                    │  snet-apps:   Container Apps / App Service VNet integration (your front end)    │
                    │  UDR 0.0.0.0/0 → Azure Firewall (egress allow-list)                              │
                    └─────────────────────────────────────────────┘
APIM (internal/injected) in front of Foundry for apps and as the AI gateway
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Public vs private | Private endpoints and disabled public access for production with sensitive data; public with Entra ID plus IP rules can be acceptable for dev or low-risk |
| BYO VNet vs managed VNet for agents | BYO VNet when you have a hub-and-spoke network team, firewall-based egress and custom DNS; managed VNet for simpler setups (check feature status) |
| Basic vs standard agent setup | Standard (BYO Storage, Cosmos DB, Search) when you need data isolation, private networking and control of agent state; basic for prototypes |
| Internet-dependent tools | Allow only required FQDNs through the firewall; avoid web grounding in highly restricted environments |
| Portal access | Plan jump boxes, AVD or VPN for portal use against private resources; some operations need the SDK or CLI |

## Hands-on

1. Deploy Microsoft's **network-isolated standard agent setup** template into a test subscription.
2. Confirm from your laptop (outside the VNet) that the project endpoint fails to resolve or connect, then succeeds from a VM in the VNet.
3. Use `nslookup <resource>.services.ai.azure.com` inside the VNet and confirm it resolves to a private IP.
4. Create an agent with a knowledge tool on the private AI Search index and run it; check that the Search service has public access disabled.
5. Add a UDR to send agent subnet egress to Azure Firewall, and allow one external API FQDN. Test a tool call that needs it.

## Security, cost and reliability notes

- **Security:** inbound private endpoints don't stop outbound data exfiltration by agents. Control egress with injection plus firewall, or a managed VNet with approved outbound.
- **Security:** validate DNS end to end. Most private networking incidents are DNS misconfigurations.
- **Cost:** private endpoints, Azure Firewall, Cosmos DB and dedicated AI Search add steady costs. Budget them in the landing zone.
- **Reliability:** the agent subnet and IP ranges are hard to change later. Size subnets per Microsoft's guidance and avoid overlapping ranges with on-premises.

## Common mistakes

- Creating private endpoints but forgetting one of the three DNS zones.
- Disabling public access and then being unable to use the portal from developer laptops.
- Securing Foundry but leaving connected Storage or Search public.
- Ignoring egress, so agents can call any internet endpoint.
- Undersized or overlapping agent subnets.

## Check yourself

1. Which private DNS zones does a Foundry resource need?
2. What does the agent subnet need to be delegated to?
3. What resources does the standard agent setup bring your own of?
4. Why don't private endpoints alone prevent data exfiltration?
5. What are the two outbound isolation models for agents?
6. How do you use the portal when public access is disabled?

<details>
<summary>Answers</summary>

1. `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com` and `privatelink.services.ai.azure.com`.
2. `Microsoft.App/environments`.
3. Azure Storage (files), Azure Cosmos DB for NoSQL (conversations/threads) and Azure AI Search (vector stores).
4. They control inbound access only; agents and tools can still make outbound calls unless egress is restricted.
5. Customer-managed VNet (network injection into your delegated subnet) and a Foundry-managed virtual network.
6. From a network that can reach the private endpoint, such as VPN or ExpressRoute, a jump box via Bastion, or Azure Virtual Desktop. Some tasks need the SDK or CLI.

</details>

## References

- [Configure private link for Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Virtual networks for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [Configure managed virtual network](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [Use your own resources with Agent Service (standard setup)](https://learn.microsoft.com/azure/foundry/agents/how-to/use-your-own-resources)
- [Azure Private Endpoint DNS zone values](https://learn.microsoft.com/azure/private-link/private-endpoint-dns)
- [Baseline Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
