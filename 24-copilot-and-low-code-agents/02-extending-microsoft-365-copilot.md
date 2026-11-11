# Extending Microsoft 365 Copilot

## Why it matters

Out of the box, Microsoft 365 Copilot knows your Microsoft 365 content and the web. It doesn't know your CRM, ticketing system, product database or the procedures of your finance team, and it can't act in those systems. **Extensibility** fills the gap: **connectors** bring external knowledge in, **declarative agents** give Copilot a focused persona with curated knowledge and actions, **MCP and API plugins** let those agents call your services, and **custom engine agents** bring your own orchestration and models (often on Azure) into Copilot and Teams. Choosing the right extension type determines cost, control, governance and how much Azure you need. It's also how Azure-built AI reaches users where they already work.

## Key concepts

### Extension types

| Type | What it is | Orchestration and model | Built with |
|---|---|---|---|
| **Copilot connector (synced)** | Ingests external content into Microsoft Graph with an `externalItem` schema; semantically indexed | Copilot's | Graph connectors API, prebuilt connectors |
| **Copilot connector (federated)** | Retrieves content at query time via **MCP** without indexing into Graph | Copilot's | MCP server |
| **Declarative agent** | Configuration: name, instructions, knowledge, capabilities, actions, conversation starters | Copilot's orchestrator and models | Agent Builder in Copilot, Copilot Studio, Microsoft 365 Agents Toolkit |
| **MCP / API plugin** | Custom action for a declarative agent: MCP server tools or REST API with OpenAPI description | Copilot's (decides when to call) | Agents Toolkit, Copilot Studio |
| **Custom engine agent** | Your own orchestration, models, workflows and proactive behaviour, surfaced in Copilot and Teams | **Yours** | Copilot Studio (low-code), Microsoft 365 Agents SDK, Teams SDK, Foundry agents |

Plugins aren't standalone experiences in Copilot: they're actions inside declarative agents.

### Declarative agents

Choose one when users need a named experience for a defined outcome, consistent instructions, curated knowledge, specific actions and conversation starters. It's not needed if an existing experience or approved agent already does the job, and it's the wrong choice if you need full control of orchestration, model, hosting or UI (use a custom engine agent).

Knowledge and capabilities can include SharePoint and OneDrive sites or files, Teams messages, email, people, connectors, web search, code interpreter, image generation, and actions (MCP/API plugins). Users of declarative agents that ground on work data need the appropriate Microsoft 365 Copilot licence or pay-as-you-go billing.

### MCP and API plugins

- **MCP plugins**: Copilot discovers the server's tools **dynamically at runtime** by default, so new tools appear without repackaging the agent.
- **API plugins**: an OpenAPI description plus a plugin manifest describing functions, confirmation prompts and response rendering (Adaptive Cards).
- Authentication: Entra ID SSO / OAuth or API key via the Teams developer portal registration.
- Write actions should require user confirmation.

### Custom engine agents

| Approach | Use |
|---|---|
| **Copilot Studio** (low-code) | Managed SaaS, built-in compliance, quick to scale (file 03) |
| **Microsoft 365 Agents SDK** | Full-stack, multi-channel agents (Copilot, Teams, web, partner channels) with any orchestrator (Agent Framework, LangChain) and any model |
| **Teams SDK** | Collaborative agents in Teams channels and meetings |
| **Foundry agents** | **Publish from Foundry** to Microsoft 365 Copilot and Teams (auto-provisions Azure Bot Service and Entra app), or integrate via a proxy app built with the Agents Toolkit for advanced customisation |

### Choosing between them

| Need | Choose |
|---|---|
| Copilot should *know* about external content | Synced connector (searchable, indexed) or federated connector (data must stay in source) |
| A focused assistant on Copilot's engine | Declarative agent |
| That assistant must *act* in your systems | Declarative agent + MCP/API plugin |
| Own model, orchestration, multi-agent workflows, proactive triggers, other channels | Custom engine agent |
| Already built in Foundry | Publish the Foundry agent to Microsoft 365 |

## How it works on Azure

Typical Azure building blocks behind extensions:

| Extension | Azure services |
|---|---|
| API plugin | Azure Functions or App Service API, API Management (auth, throttling), Entra app registration |
| MCP plugin / federated connector | MCP server on Azure Functions, Container Apps or App Service, fronted by API Management |
| Synced connector | Azure Functions or Logic Apps job reading Azure SQL, Storage or SaaS and pushing items to Graph |
| Custom engine agent | Foundry Agent Service or Agent Framework on Container Apps/App Service; Azure Bot Service for channels |

Declarative agent manifest sketch (built with the Agents Toolkit; check the current schema version):

```json
{
  "name": "Travel Budget Assistant",
  "description": "Answers questions about team travel budgets and records approved expenses.",
  "instructions": "Help employees check travel budgets. Only charge expenses after the user confirms the amount and budget. Cite the policy page when explaining rules.",
  "capabilities": [
    { "name": "OneDriveAndSharePoint", "items_by_url": [ { "url": "https://contoso.sharepoint.com/sites/Finance/TravelPolicy" } ] }
  ],
  "actions": [ { "id": "budgetsPlugin", "file": "budgets-plugin.json" } ],
  "conversation_starters": [ { "text": "How much is left in the Contoso travel budget?" } ]
}
```

Publishing a Foundry agent: in the Foundry portal, **Publish → Microsoft 365 Copilot and Teams**, which creates the Bot Service resource and Entra registration; then an admin approves it in the Microsoft 365 admin centre ([Topic 18 file 12](../18-agents-with-foundry-agent-service/12-publishing-agents.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Declarative first** | Start with a declarative agent if Copilot's orchestration suffices; less to build and operate, inherits Copilot's security and compliance. |
| **Connector type** | Synced for broad search and semantic retrieval; federated when data can't leave the source or must be real time. |
| **Custom engine when** | You need a specific model, deterministic workflows, multi-agent orchestration, proactive messages, non-Microsoft channels, or your own evaluation and guardrails. |
| **Identity** | SSO with on-behalf-of so actions respect the user's permissions. |
| **Licensing** | Factor Copilot licences or pay-as-you-go for users of work-data agents; custom engine agents also incur Azure costs. |
| **Governance** | Admin approval for org-wide publishing; register in Agent 365 (file 06). |

## Hands-on

1. Create a declarative agent in Agent Builder grounded on one SharePoint site; test instructions and starters.
2. Rebuild it with the Agents Toolkit and add an MCP plugin pointing to an Azure Functions MCP server behind API Management.
3. Add a confirmation step for the write tool.
4. Create a federated connector over an MCP server for a small dataset; compare with a synced connector.
5. Publish an existing Foundry agent to Microsoft 365 Copilot and Teams; complete admin approval.

## Security, cost and reliability notes

- **Plugins and MCP servers are attack surface**: authenticate, validate inputs, least privilege, rate limit.
- **Data leaving Microsoft 365**: plugins send user queries and context to your service; document data flows and protect them.
- **Indirect injection**: content returned from plugins and connectors enters Copilot's context; sanitise and minimise.
- **Reliability**: Copilot calls your APIs at user scale; monitor latency and errors.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building a custom engine agent when a declarative agent would do | Start declarative |
| Synced connector for highly regulated data that must stay in place | Federated connector |
| API plugins with shared service credentials | SSO/OBO |
| No confirmation on write actions | Require confirmation |
| Publishing without admin governance | Admin approval and registry |

## Check yourself

1. Compare synced and federated connectors.
2. What does a declarative agent consist of?
3. How do MCP plugins differ from API plugins?
4. When do you need a custom engine agent?
5. What are the two ways to bring a Foundry agent into Microsoft 365?
6. Which Azure services typically host an MCP plugin?

<details>
<summary>Answers</summary>

1. Synced: ingest into Graph with schema, semantically indexed. Federated: query-time retrieval via MCP, no data movement or indexing.
2. Identity and behaviour (name, instructions, starters), knowledge sources, capabilities and actions, and host/distribution metadata.
3. MCP plugins call MCP server tools discovered dynamically at runtime; API plugins call REST APIs described with OpenAPI and a plugin manifest.
4. When you need your own orchestration, models, workflows, proactive triggers or channels beyond Copilot's engine.
5. Publish directly from the Foundry portal (auto-provisions Bot Service and Entra), or a proxy app built with the Microsoft 365 Agents Toolkit.
6. Azure Functions, Container Apps or App Service, usually fronted by API Management with Entra authentication.

</details>

## References

- [Microsoft 365 Copilot extensibility](https://learn.microsoft.com/microsoft-365-copilot/extensibility/)
- [Declarative agents](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-declarative-agent)
- [Custom engine agents](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-custom-engine-agent)
- [MCP and API plugins](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-api-plugins)
- [Copilot connectors](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-copilot-connector)
