# Copilot Studio with Foundry

## Why it matters

Low-code and pro-code aren't rivals; the best enterprise solutions combine them. Business teams move fast in Copilot Studio with connectors, topics and channels, while AI engineers build what low-code can't do well in Microsoft Foundry: custom or fine-tuned models, complex multi-agent logic, rigorous evaluation, hosted code. Copilot Studio can **bring Foundry models** into prompts, **call Foundry agents** as connected agents, use **Azure AI Search** for knowledge, and call **Azure-hosted MCP servers and APIs**. Foundry agents can in turn be published to Microsoft 365 Copilot and Teams. Knowing these integration points, and their preview status and limits, lets architects design a "fusion" model where each team works in the tool that suits it.

## Key concepts

### Integration patterns

| Pattern | Direction | Use | Status |
|---|---|---|---|
| **Foundry models in the Prompt tool** (bring your own model) | Copilot Studio → Foundry model deployment | Use a specific, fine-tuned or industry model for a prompt step (classification, extraction, summarisation) | Available in Copilot Studio, Power Apps and Power Automate; check supported model families |
| **Connected Foundry agent** | Copilot Studio agent → Foundry agent | Delegate complex reasoning, specialised tools or multi-agent logic to a pro-code agent | **Preview** |
| **Azure AI Search knowledge** | Copilot Studio → Search index | Large, curated, vector/hybrid knowledge built by data engineers | GA |
| **MCP / REST tools on Azure** | Copilot Studio → Functions/Container Apps via APIM | Reusable enterprise actions shared with Foundry agents | GA (MCP needs generative orchestration) |
| **Publish Foundry agent to Microsoft 365** | Foundry → Copilot and Teams | Pro-code agent surfaced where users work ([Topic 18 file 12](../18-agents-with-foundry-agent-service/12-publishing-agents.md)) | See Foundry docs |
| **A2A** | Agent ↔ agent | Cross-platform agent collaboration ([Topic 19](../19-agent-frameworks-and-protocols/README.md)) | Varies |

### Bring your own model for prompts

In Copilot Studio: **Tools → New tool → Prompt**, then in the model dropdown select **+** to connect a model from Foundry, entering the deployment name and base model name exactly as in Foundry, the endpoint and authentication. The prompt can then be used by agents and flows. Microsoft's page currently lists model-family limitations (for example, some GPT-5-family models weren't supported for this feature at the time of writing): check before designing around a specific model.

### Connecting a Foundry agent (Preview)

Requirements and steps from Microsoft's documentation:

1. The Foundry agent must have the **Activity protocol endpoint** enabled. New Foundry agents expose only the **Responses** and **A2A** endpoints by default; enable Activity programmatically (REST API or Python SDK). The portal's Endpoints section keeps showing only Responses and A2A even after enabling, which is expected.
2. Only agents created in the **new Foundry portal** are supported (older ones fail with "404 - Version not found").
3. In Copilot Studio: **Agents → Add an agent → Connect to an external agent → Microsoft Foundry**, select or create a connection using the Foundry **project endpoint**, then enter a name, a precise description (used by the orchestrator to decide when to call it) and the **Agent ID**.
4. If calls fail with "endpoint does not support activity", the Activity endpoint isn't enabled.

You're responsible for data flows, quality, permissions, observability and human oversight across connected agents.

### Division of labour

| Concern | Copilot Studio (low-code) | Foundry (pro-code) |
|---|---|---|
| Channels and UX | Teams, Microsoft 365 Copilot, web, telephony | API and custom apps; publish to Microsoft 365 |
| Business process | Topics, agent flows, 1,000+ connectors | Code, hosted agents, workflows |
| Models | Managed models plus Foundry models via prompts | Full catalogue, fine-tuning, routing, provisioned throughput |
| Knowledge | SharePoint, Dataverse, Search, files | Foundry IQ, Azure AI Search, custom RAG |
| Evaluation | Test sets, graders, analytics | Full evaluator suite, CI gates, red teaming ([Topic 22](../22-evaluation-observability-and-genaiops/README.md)) |
| Governance | Power Platform admin, DLP, environments | Azure RBAC, Policy, Foundry Control Plane |
| Ownership | Business makers, CoE | AI engineering team |

## How it works on Azure

Reference fusion architecture:

```text
Teams / Microsoft 365 Copilot / website
          │
   Copilot Studio agent (topics, connectors, human hand-off, analytics)
     ├── Knowledge: Azure AI Search index (built by data team)
     ├── Prompt tool: fine-tuned classifier model deployed in Foundry
     ├── MCP tool: Order API MCP server (Container Apps) ── API Management ── ERP
     └── Connected agent: Foundry "claims analyst" agent (hosted, with code interpreter and evaluation)
                              └── Foundry project: tracing → Application Insights, guardrails, evaluations
```

Enabling the Activity endpoint is done with the Foundry agents API; follow "Configure and share your agent" in the Foundry documentation for the current request shape, since it's a programmatic-only setting.

Shared foundations:

- **One API Management gateway** for models, MCP servers and APIs used by both platforms (token limits, auth, logging) ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)).
- **Entra** for user SSO and agent identities across both.
- **Purview and Defender** covering both Microsoft 365 and Azure (Topic 23).
- **Agent 365** registry for agents from both platforms (file 06).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Where the "brain" lives** | Copilot Studio for conversational front door and simple logic; Foundry for complex reasoning that needs pro-code evaluation and control. |
| **Preview dependency** | Connected Foundry agents are Preview; for production, consider calling Foundry via a custom connector or MCP tool until GA. |
| **Model choice** | Use BYOM prompts only where a managed model doesn't meet quality or cost needs; check supported families. |
| **Tool reuse** | Build MCP servers once on Azure; reuse from both Copilot Studio and Foundry. |
| **Observability** | Correlate Copilot Studio analytics with Foundry traces using conversation IDs passed between them. |
| **Ownership boundaries** | Define contracts (input/output schemas, SLAs) between maker and engineering teams. |

## Hands-on

1. Deploy a model in Foundry and use it in a Copilot Studio prompt tool for ticket classification.
2. Create a Foundry prompt agent in the new portal; enable its Activity endpoint via SDK or REST.
3. Connect it to a Copilot Studio agent with a clear description; test when the orchestrator delegates.
4. Add an Azure AI Search index as knowledge and an MCP tool on Azure Functions shared with the Foundry agent.
5. Trace a request end to end: Copilot Studio analytics plus Foundry traces.

## Security, cost and reliability notes

- **Identity across boundaries**: decide whether the Foundry agent acts as itself or on behalf of the user; avoid broad shared keys in connections.
- **Data flows**: conversation context passes from Copilot Studio to Foundry; document and approve.
- **Cost**: both Copilot Studio consumption and Azure consumption (models, Search, hosting) apply.
- **Latency**: each hop (Copilot Studio → Foundry → tools) adds latency; keep connected agents focused.
- **Preview** features can change; pin designs to documented behaviour and test after updates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rebuilding the same tool separately in both platforms | Shared MCP servers behind APIM |
| Connecting a Foundry agent without enabling Activity | Enable the Activity endpoint first |
| Using an agent from the old Foundry portal | Recreate in the new portal |
| Vague connected-agent descriptions | Precise purpose so the orchestrator delegates correctly |
| Two separate governance worlds | Agent 365 registry, shared gateway, Purview and Defender |

## Check yourself

1. List four ways Copilot Studio and Foundry integrate.
2. How do you use a Foundry model in Copilot Studio?
3. What are the prerequisites for connecting a Foundry agent?
4. Which endpoints does a new Foundry agent expose by default?
5. How should responsibilities be split between Copilot Studio and Foundry?
6. Why share MCP servers between the two?

<details>
<summary>Answers</summary>

1. Foundry models in the Prompt tool, connected Foundry agents (Preview), Azure AI Search knowledge, MCP/REST tools on Azure, publishing Foundry agents to Microsoft 365, A2A.
2. Prompt tool → model dropdown → + connect a Foundry model with deployment name, base model name, endpoint and authentication.
3. Activity protocol endpoint enabled programmatically, agent created in the new Foundry portal, a connection using the project endpoint, and the agent ID.
4. Responses and A2A.
5. Copilot Studio: channels, conversational UX, connectors, simple logic. Foundry: complex reasoning, custom models, evaluation, hosted code.
6. Build and secure actions once, govern them centrally through APIM, and keep behaviour consistent across platforms.

</details>

## References

- [Connect to a Microsoft Foundry agent (Preview)](https://learn.microsoft.com/microsoft-copilot-studio/add-agent-foundry-agent)
- [Bring your own model for prompts](https://learn.microsoft.com/microsoft-copilot-studio/bring-your-own-model-prompts)
- [Extend your agent with MCP](https://learn.microsoft.com/microsoft-copilot-studio/agent-extend-action-mcp)
- [Custom engine agents (Foundry integration)](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-custom-engine-agent)
- [Knowledge sources in Copilot Studio](https://learn.microsoft.com/microsoft-copilot-studio/knowledge-copilot-studio)
