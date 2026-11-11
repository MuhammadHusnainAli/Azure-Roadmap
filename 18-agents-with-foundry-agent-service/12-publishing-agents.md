# Publishing Agents

## Why it matters

An agent that only works in the Foundry playground delivers no value. Users live in Microsoft Teams, Microsoft 365 Copilot, web apps and line-of-business systems; other agents increasingly call agents through standard protocols. Publishing decides **where** an agent appears, **who** can call it, **which version** they get, and **which identity** it uses once it leaves the project. In late 2026 Microsoft simplified the model: every new agent has a stable endpoint and identity from creation, and "publishing" now mostly means distributing to Microsoft 365 and Teams. Architects need to understand endpoint authorisation and data flows; consultants need to explain the Teams/Copilot rollout and admin approval process.

## Key concepts

### The new agent object model

| | Legacy (Agent Applications) | Current model |
|---|---|---|
| Objects | Agent (data plane) + Agent Application + Deployment (control plane) | **Agent only** |
| Stable endpoint | Created when publishing an application | **Every agent, at creation** |
| Identity | Unpublished agents share the project's identity; published apps get their own | **Each new agent has its own Entra agent identity and blueprint** |
| Version routing | Deployment routes 100% to one version | **Version selector** on the agent endpoint |
| "Publish" means | Create application + deployment | Choose active version; distribute to Microsoft 365 and Teams |

Existing Agent Applications keep working; to modernise, create a new agent with the same definition (there's no in-place identity upgrade yet) and repoint clients.

Endpoint URLs:

| Protocol | Current endpoint |
|---|---|
| Responses (OpenAI-compatible) | `https://{account}.services.ai.azure.com/api/projects/{project}/agents/{agent}/endpoint/protocols/openai/responses` |
| Activity (Bot Service, Teams) | `.../agents/{agent}/endpoint/protocols/activityprotocol` |
| A2A | `.../agents/{agent}/endpoint/protocols/a2a` |
| Invocations (hosted agents) | `.../agents/{agent}/endpoint/protocols/invocations` |

### Choosing the active version

Set **Always use latest** or pin a version on the agent's `version_selector` (rule type `FixedRatio` with `traffic_percentage`). Pin in production and promote deliberately after evaluation.

### Endpoint authorisation schemes

| Scheme | How access is decided | Use for |
|---|---|---|
| **AzureRbac** (formerly `Entra`) | Token for `https://ai.azure.com` + Azure RBAC (`.../endpoints/interact/action`, for example **Foundry Agent Consumer**), at project or **agent scope** | Apps, SDKs, services |
| **AgentBlueprint** (Preview) | Token for the agent's blueprint; validation rules on the scheme | Calling one agent without Foundry RBAC |
| **BotServiceRbac** | Azure Bot Service validates the channel; Azure RBAC per user | Teams/M365 where each user needs a role ("Just you"/shared scope) |
| **BotServiceTenant** | Bot Service validates; any signed-in user in the tenant | Tenant-wide Teams/M365 distribution |
| **BotService** | Legacy | Don't use for new agents |

**API keys aren't supported** for agent endpoints. You can combine schemes (for example AzureRbac for apps and BotServiceTenant for Teams), but only one Bot Service scheme at a time.

### Channels

| Channel | How |
|---|---|
| **Microsoft 365 Copilot and Teams** | Portal **Publish → Teams and Microsoft Copilot**: Foundry builds the Teams app manifest, submits it to the catalogue, enables the Activity protocol and sets the Bot Service auth scheme. No separate Azure Bot resource needed. |
| **Web and custom apps** | Call the Responses endpoint with Entra tokens; hosted agents can publish a preview web app when Responses is supported |
| **Other agents (A2A)** | Enable **incoming A2A** (Preview): publish an **agent card** and enable the A2A protocol on the endpoint (REST; A2A 1.0 GA protocol, 0.3 preview) |
| **Microsoft Agent 365** | Published Foundry agents sync to the Agent 365 registry; hosted agents can become **autopilots** with their own Entra agent user account |
| **Copilot Studio** | Connect Foundry IQ knowledge or call agents from Copilot Studio ([Topic 24](../Roadmap.md)) |
| **Telephony** | Voice agents via Teams Phone extensibility or Twilio (file 11) |

### Microsoft 365 publishing scopes

| Portal option | REST `publishScope` | Auth scheme | Admin approval | Visibility |
|---|---|---|---|---|
| **Just you** | `Shared` | BotServiceRbac | Not required | Under *Your agents*; share the link; users need Foundry permissions |
| **People in your organisation** | `Tenant` | BotServiceTenant | **Required** in the Microsoft 365 admin centre | *Built by your org* for all tenant users |

Metadata: name, semantic version (major.minor.patch), short and long description, developer, optional website, terms and privacy URLs. Don't put secrets in metadata.

Data note from Microsoft Learn: when you publish to Microsoft 365 and Teams, those services process and store publishing data and agent responses under **Microsoft 365's** terms, compliance, residency and data handling commitments. Evaluate this before publishing.

Projects with public network access disabled can't publish from the portal; use the REST API with the source-IP-filtered public Activity Protocol route.

## How it works on Azure

Pin version 2 as the active version:

```http
PATCH {project_endpoint}/agents/{agent_name}?api-version=v1
Content-Type: application/merge-patch+json
Authorization: Bearer {token}

{ "agent_endpoint": { "version_selector": { "version_selection_rules": [
    { "type": "FixedRatio", "agent_version": "2", "traffic_percentage": 100 } ] } } }
```

Grant a client app access to **one** agent only by assigning Foundry Agent Consumer at agent scope rather than project scope.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Teams/M365 vs custom web app** | Teams/Copilot when users work there and you want zero front-end build; custom app for branded experiences, anonymous users or complex UI. |
| **Just you vs organisation** | Pilot with "Just you", then go tenant-wide through admin approval with a support and feedback plan. |
| **Auth scheme** | AzureRbac for internal apps with agent-scope roles; BotServiceTenant for tenant-wide Teams; AgentBlueprint (Preview) only when you must avoid Foundry RBAC for callers. |
| **Version strategy** | Pin and promote; never "always latest" for broad audiences. |
| **A2A exposure** | Expose agents as A2A endpoints when other agents (in Foundry or elsewhere) should delegate to them; publish a precise agent card. |
| **Identity after publishing** | Tools using agent identity need RBAC for that agent's identity (it differs from the project identity on legacy agents). |
| **Governance** | Register and monitor agents in Agent 365 / Entra; apply access reviews and lifecycle policies. |

## Hands-on

1. Take the `policy-helper` agent from file 03. Pin the active version to the evaluated one.
2. **Publish → Teams and Microsoft Copilot → Just you**. Fill in metadata, publish, open Teams and chat with the agent.
3. Assign a colleague **Foundry Agent Consumer** at **agent scope**; confirm they can call the agent via its endpoint but can't edit it.
4. Enable incoming A2A with an agent card (REST PATCH per the Microsoft Learn guide), then call it from another agent with the A2A tool (file 09).
5. Prepare a tenant-wide rollout checklist: admin approval request, support contact, privacy statement URL, monitoring dashboard, rollback plan (switch version selector).

## Security, cost and reliability notes

- **Data residency**: Microsoft 365 and Teams processing follows Microsoft 365 commitments, not just your Azure region.
- **No API keys**: use Entra ID; keyless by design.
- **Least privilege**: agent-scope roles for callers; agent identity roles for tools.
- **Admin approval** is a governance checkpoint; provide clear descriptions and owners.
- **Change management**: version pinning plus evaluation gates (file 03, [Topic 16 file 04](../16-prompt-engineering-and-model-customisation/04-prompt-management.md)).
- **Cost**: broader distribution multiplies usage; set budgets and monitor per-agent consumption.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new Agent Applications | Use the agent endpoint model |
| Expecting API keys for endpoints | Use Entra tokens and RBAC |
| Tenant-wide publish without admin engagement | Plan the approval and support process |
| Forgetting that publish metadata is visible to users | No secrets or internal details |
| Portal publishing from a fully private project | Use the REST path with the filtered Activity route |
| "Always latest" for production users | Pin versions |

## Check yourself

1. What changed between the legacy Agent Application model and the current agent object model?
2. What are the Responses and A2A endpoint paths for an agent?
3. Compare AzureRbac, BotServiceRbac and BotServiceTenant.
4. What does Foundry do when you publish to Teams and Microsoft Copilot?
5. What's the difference between "Just you" and "People in your organisation"?
6. Which data-handling commitments apply after publishing to Microsoft 365?
7. How do you expose an agent to other agents?
8. How would you let an app call exactly one agent?

<details>
<summary>Answers</summary>

1. Applications and deployments were folded into the agent: every new agent has a stable endpoint, its own Entra identity and blueprint, and a version selector; publishing now means distribution to Microsoft 365/Teams.
2. `.../agents/{agent}/endpoint/protocols/openai/responses` and `.../agents/{agent}/endpoint/protocols/a2a`.
3. AzureRbac: Foundry-scoped token plus Azure RBAC for apps; BotServiceRbac: channel validated by Bot Service plus per-user RBAC; BotServiceTenant: any signed-in user in the tenant via Bot Service.
4. Validates metadata, builds a Teams app manifest, submits it to the Copilot and Teams catalogues, enables the Activity protocol and sets the Bot Service authorisation scheme.
5. "Just you" (Shared, BotServiceRbac) is immediate and personal; "People in your organisation" (Tenant, BotServiceTenant) requires Microsoft 365 admin approval and appears for the whole tenant.
6. Microsoft 365 and Teams terms, compliance, data residency and data handling commitments.
7. Enable incoming A2A: publish an agent card and enable the A2A protocol on the agent endpoint.
8. Assign Foundry Agent Consumer (or the interact permission) at the agent's scope rather than the project.

</details>

## References

- [Publish agents to Microsoft Copilot and Teams](https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot)
- [Migrate from agent applications to the new agent model](https://learn.microsoft.com/azure/foundry/agents/how-to/migrate-agent-applications)
- [Agent endpoint authorisation](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-endpoint-authorization)
- [Enable incoming A2A (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint)
- [Microsoft Agent 365 integration with Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration)
- [Agent applications (legacy)](https://learn.microsoft.com/azure/foundry/agents/how-to/agent-applications)
