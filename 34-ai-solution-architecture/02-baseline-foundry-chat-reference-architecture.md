# Baseline Foundry Chat Reference Architecture

## Why it matters

Most enterprise AI programmes start with a chat assistant grounded on company data, and most of them reinvent the same secure foundation: private networking, identity, content safety, zone redundancy, observability and safe deployment of agents. Microsoft's **baseline Microsoft Foundry chat reference architecture** in the Azure Architecture Center is the canonical, Well-Architected design for that scenario, with a reference implementation and a variant for **Azure landing zones**. Understanding it component by component, including its explicit trade-offs and the responsibilities it places on you, is the fastest route to a production-grade AI design and a common interview and exam topic.

## Key concepts

### Request flow

1. A user opens the chat UI; traffic enters through **Application Gateway** with **Web Application Firewall** (TLS termination, certificates in Key Vault).
2. The web app on **App Service** (zone-redundant, VNet-integrated) invokes the agent using **Microsoft Agent Framework**, calling the server-side agent definition in **Foundry Agent Service**.
3. The agent applies its system prompt, its configured **model** and **tools**: the **Azure AI Search** tool for grounding over a private endpoint; other tools (public MCP servers, custom APIs) egress through **Azure Firewall**; private MCP servers/APIs stay in the VNet.
4. The agent calls the model and returns the response.
5. The conversation (requests, responses, tool invocations) is persisted in **Cosmos DB** as the agent's memory; the Foundry APIs support multiple concurrent, context-isolated conversations.

### Components

| Component | Role |
|---|---|
| **Foundry Agent Service (standard setup)** | Managed runtime for prompt agents; you supply dedicated **Cosmos DB** (conversation/state, database `enterprise_memory`), **Storage** (uploaded files) and **AI Search** (file search index) that the service manages exclusively |
| **Foundry models** | Model deployments (Data Zone provisioned + Data Zone standard with spillover recommended for enterprise chat; Global if residency allows) |
| **App Service** | Hosts the chat UI/API; zone-redundant; code deployed from Storage (run-from-package) |
| **Application Gateway + WAF** | Secure entry point |
| **Azure AI Search** | Enterprise knowledge index (separate from the agent's file-search dependency) |
| **Azure Firewall** | Inspects and controls egress with FQDN rules |
| **Private Link + private DNS zones** | Private connectivity to all PaaS services |
| **Key Vault** | TLS certificates |
| **Azure Monitor / Application Insights** | Observability and tracing |
| **Azure Policy, Defender for Cloud, Purview** | Governance, threat protection and data security |

### Key considerations from the architecture

| Area | Guidance |
|---|---|
| **Zone redundancy** | Use zone-redundant Cosmos DB, Storage (ZRS) and AI Search (3 replicas) for agent dependencies; App Service with 3+ instances; consider Cosmos DB global distribution |
| **Model hosting reliability** | Standard deployments are single-region without zones; combine Data Zone provisioned + standard with spillover; Foundry doesn't provide round-robin or circuit breaking, so put a gateway in front of models if you need it (file 05) |
| **Isolate agent dependencies** | Don't share the Agent Service's Cosmos DB/Storage/Search with other workload components |
| **Conversation isolation** | Agent Service doesn't enforce per-user authorisation on conversations; your app must verify the authenticated user owns the conversation ID and never trust client-supplied IDs |
| **Identity** | Managed identities everywhere; least privilege per component |
| **Content safety** | Guardrails on models and agents; Prompt Shields (Topic 23 file 03) |
| **Purview** | Data security and compliance for AI interactions |
| **Agent lifecycle** | Agents as code; automated deployment from CI/CD via network-connected build agents; independent agent pipelines loosely coupled from UI code; statically defined agents unless dynamic behaviour is required |
| **Disaster recovery** | Foundry doesn't provide automatic DR; plan recovery of agent state and redeploy from code (Topic 28 file 06) |
| **Cost** | Agent Service dependencies, models (PTU vs standard), firewall and private networking, Defender plans |

### Landing zone variant

The **baseline Foundry chat in an Azure landing zone** splits responsibilities: the **platform team** provides hub networking (firewall, DNS, connectivity), policies and shared services; the **workload team** deploys the application landing zone (spoke VNet, App Service, Foundry, AI Search, Cosmos DB, Storage) and requests firewall rules and DNS entries from the platform team.

## How it works on Azure

Conversation ownership check in the app tier (pseudocode):

```python
def send_message(user, conversation_id, text):
    conv = conversations_index.get(conversation_id)     # your own mapping: conversation -> owner
    if conv is None or conv.owner_oid != user.oid:
        raise PermissionError("Conversation not found")   # never forward unverified IDs
    with project.get_openai_client(agent_name="hr-assistant") as client:
        return client.responses.create(conversation=conversation_id, input=text)
```

Deployment path: Bicep/Terraform for infrastructure (from the reference implementation), agent definitions in Git deployed by pipeline (Topic 27 file 06), evaluation gates and red teaming before production (Topic 22).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Basic vs baseline** | Basic architecture for learning/PoC; baseline for production with network isolation and zone redundancy. |
| **Prompt agent vs hosted agent** | Prompt agents (baseline) for configuration-driven agents; hosted agents for code-heavy logic and per-user session isolation when user identity is passed. |
| **Single vs multi-agent** | Start single-agent; add multi-agent orchestration only when justified (file 04). |
| **Client-managed chat history** | Alternative to service-managed conversations when you need full control of storage. |
| **Model deployment types** | Data Zone for EU/US processing constraints; Global for resilience and cost when allowed. |
| **Multi-region** | Add when RTO/regional resilience requires it (file 07). |

## Hands-on

1. Read the baseline article end to end and redraw the architecture with numbered flows.
2. Deploy the reference implementation to a sandbox; trace a request in Application Insights.
3. Implement and test the conversation ownership check.
4. Configure Data Zone provisioned + standard deployments with spillover.
5. Map responsibilities to platform vs workload teams for the landing zone variant.

## Security, cost and reliability notes

- **Egress control**: all tool calls to the internet through the firewall with FQDN allow-lists.
- **Conversation isolation** is your responsibility.
- **Cost**: firewall, private endpoints, zone-redundant dependencies and PTUs make the baseline expensive; scale down for non-production.
- **Reliability**: zone-redundant dependencies and model spillover; DR plan for agent state.

## Common mistakes

| Mistake | Fix |
|---|---|
| Trusting client-supplied conversation IDs | Ownership checks |
| Sharing Agent Service dependencies with other components | Dedicated resources |
| Single standard model deployment in production | Provisioned + spillover or gateway |
| Editing agents in the portal | Agents as code |
| Public endpoints left enabled | Private Link and policy |

## Check yourself

1. Describe the request flow of the baseline.
2. Which dependencies does Agent Service standard setup require, and who owns them?
3. How does the baseline handle model reliability?
4. What is the conversation isolation responsibility?
5. How should agents be deployed?
6. How does the landing zone variant split responsibilities?

<details>
<summary>Answers</summary>

1. User → Application Gateway/WAF → App Service → Agent Framework → Foundry agent → tools (AI Search privately, external via firewall) → model → response; conversation stored in Cosmos DB.
2. Cosmos DB (state/conversations), Storage (files), AI Search (file search index); you provision them in your subscription, Agent Service manages them exclusively.
3. Data Zone provisioned plus standard with spillover (Global where allowed); a gateway for advanced load balancing or circuit breaking.
4. The app must verify the authenticated user owns the conversation ID because Agent Service doesn't enforce per-user authorisation.
5. As code from source control via CI/CD pipelines using network-connected build agents, independently deployable from UI code.
6. Platform team: hub networking, firewall, DNS, policy; workload team: spoke and application resources, requesting platform changes.

</details>

## References

- [Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
- [Baseline Foundry chat in an Azure landing zone](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone)
- [Gateway in front of multiple Azure OpenAI deployments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend)
- [Foundry Agent Service disaster recovery](https://learn.microsoft.com/azure/foundry/how-to/agent-service-disaster-recovery)
