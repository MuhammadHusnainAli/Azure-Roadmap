# Foundry Control Plane

> Topic: 14 · Microsoft Foundry Platform · Tracks: E ○ A ● C ● · Status checked: 2026-11

## Why it matters

Organisations are moving from a few copilots to fleets of agents across many projects, built on Foundry, Copilot Studio and third-party frameworks. Without central oversight nobody knows how many agents exist, what they cost, whether they're safe, or which ones break policy. **Microsoft Foundry Control Plane** is the fleet-management layer: inventory, observability, compliance, security and quota for agents, models and tools across a subscription. Architects design for it; consultants include it in every AI governance conversation.

## Key concepts

### What Control Plane provides

Microsoft describes it as "a unified management interface that provides visibility, governance, and control for AI agents, models, and tools across your Foundry enterprise". It's accessed in the Foundry portal through **Operate** (fleet-wide) and **Manage** (current project and resource).

| Pane | Purpose | Examples |
|---|---|---|
| **Overview** | Fleet health at a glance | Active agents, run completion rate, cost trends, prevented behaviours, alerts, health scores |
| **Assets** | Inventory of agents, models and tools across projects in a subscription | Filter by version, tags, health score, cost, token usage, alerts; drill into evaluation and monitoring; inline recommendations |
| **Compliance** | Define, apply and monitor guardrail and compliance policies | Guardrail policies for model deployments, integration with Azure Policy, Defender and Purview, bulk remediation, versioned policy assignments |
| **Monitoring / quality** | Continuous evaluation and observability | Task adherence, intent resolution, tool call success, groundedness, sensitive data leakage, jailbreak and cross-domain prompt injection (XPIA) exposure |
| **Security** | Red teaming and threat signals | Scheduled AI Red Teaming Agent scans, drift monitoring, Defender and Purview alerts on the dashboard |
| **Quota** (under Manage) | View, adjust and request model quota | Per-deployment usage, "Show all" for available models and regions |
| **AI Gateway** (under Manage) | Associate API Management as the AI gateway | Token limits for deployments, registering external agents and MCP tools |

Some capabilities are in preview, and the features are currently portal-only. Check the article's preview markers.

### Multi-platform agent fleet

Control Plane can show agents from several platforms (check the **supported agent platforms** list):

| Source | How it appears |
|---|---|
| Foundry Agent Service agents (prompt, workflow, hosted) | Automatically, across projects in the subscription |
| **Custom agents** running elsewhere (Azure, other clouds, on-premises; any framework) | **Registered** through the AI gateway, so traffic flows through API Management and telemetry is collected |
| MCP tools hosted anywhere | Registered via the AI gateway for governance and discovery |
| Microsoft agents (for example Copilot Studio) | Per supported-platform list; Microsoft 365-side governance is in Agent 365 |

### Prerequisites

| Requirement | Why |
|---|---|
| A Foundry project | Starting point |
| **AI gateway configured** (API Management associated with the Foundry resource) | Needed for advanced governance: token limits, registering custom agents and tools |
| Application Insights connected to projects | Traces and metrics for monitoring and evaluation |
| Appropriate RBAC at subscription scope | Fleet views span projects |

### Control Plane vs Agent 365 vs other governance

| Tool | Scope | Focus |
|---|---|---|
| **Foundry Control Plane** | Azure: Foundry resources, projects, agents, models, tools | Build-to-production lifecycle: inventory, quality, safety, cost, compliance for AI assets |
| **Microsoft Agent 365** | Microsoft 365: agents as part of the workforce | Agent identity, access, lifecycle and usage across Microsoft 365 (integrates with Foundry agents) |
| **Microsoft Entra Agent ID** | Identity for agents | Agent identities, Conditional Access, access reviews |
| **Azure Policy** | Azure resource configuration | Enforce settings (disable local auth, private endpoints, allowed models) |
| **Microsoft Purview** | Data security and compliance | DSPM for AI, DLP, audit, eDiscovery for AI interactions |
| **Microsoft Defender for Cloud** | Threat protection | AI threat protection alerts (jailbreak, data exfiltration attempts) |
| **API Management AI gateway** | Runtime traffic | Token limits, routing, logging for models, agents and MCP tools |

Control Plane brings signals from these into one view; it doesn't replace them.

## How it works on Azure

### Enabling fleet governance (high level)

```text
1. Create or choose an API Management instance (Standard v2 or another supported tier)
   → Foundry portal: Manage → AI Gateway → associate with the Foundry resource
2. Connect Application Insights to each project (tracing) and enable continuous evaluation for key agents
3. Operate → Assets: review discovered agents, models, tools; tag owners and environments
4. Register external agents (e.g. a LangGraph agent on AKS) and MCP tools through the gateway
5. Operate → Compliance: create a guardrail policy (e.g. required content filter levels, Prompt Shields on)
   and assign it to model deployments across the subscription; remediate non-compliant ones in bulk
6. Schedule AI Red Teaming Agent scans for production agents; route Defender for Cloud AI alerts
7. Set alerts on cost and token anomalies, error rates and evaluation score drops
```

### Complementary Azure Policy guardrails (examples)

| Policy intent | Example |
|---|---|
| No key-based auth | Audit or deny Cognitive Services accounts where `disableLocalAuth` isn't true |
| Private networking | Deny public network access on Foundry resources in production subscriptions |
| Approved models only | Restrict which model names and versions can be deployed (built-in or custom policy on deployments) |
| Approved deployment types | Deny Global deployments where data residency requires Data Zone or Regional |
| Diagnostic settings | DeployIfNotExists diagnostic settings to Log Analytics |

Azure Policy enforces **configuration**; Control Plane monitors **behaviour and quality** of the AI assets.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| When to adopt | As soon as more than a handful of agents or several teams exist; retrofitting governance is harder |
| Gateway placement | One AI gateway per environment or region, owned by the platform team, shared by projects |
| Custom agents | Register every production agent, whatever its framework, so inventory and cost are complete |
| Policy approach | Start in audit mode, measure non-compliance, then enforce |
| Ownership | Every asset needs an owner tag and an accountable team; Control Plane surfaces it but can't create accountability |

## Hands-on

1. Associate an API Management instance as the AI gateway for a Foundry resource.
2. Create two projects with one agent each; confirm both appear in **Operate → Assets** with health and cost data.
3. Register an external agent (for example a simple FastAPI agent on Container Apps) through the gateway and confirm it appears in the inventory.
4. Create a guardrail policy requiring specific content filter settings, assign it, deliberately misconfigure a deployment, and use bulk remediation.
5. Enable continuous evaluation on one agent and set an alert on a groundedness score drop.
6. Schedule an AI Red Teaming Agent scan and review the findings.

## Security, cost and reliability notes

- **Security:** Control Plane is a high-privilege view. Restrict fleet-wide roles to platform, security and governance teams.
- **Security:** registering custom agents through the gateway also lets you enforce authentication and token limits on them.
- **Cost:** API Management, Application Insights ingestion, continuous evaluations and red-teaming runs have their own costs. Sample traces and schedule evaluations sensibly.
- **Reliability:** Control Plane observes; it isn't in the request path for Foundry-native agents. The AI gateway, though, is in the path for traffic routed through it, so make it resilient.

## Common mistakes

- Treating Control Plane as optional "later" tooling after dozens of agents are already live.
- Not registering agents built outside Foundry, leaving blind spots in inventory and cost.
- Confusing Control Plane (Azure fleet governance) with Agent 365 (Microsoft 365 workforce governance).
- Enforcing guardrail policies without an audit phase, breaking existing deployments.
- No owner tags, so alerts have nowhere to go.

## Check yourself

1. What four areas does Foundry Control Plane consolidate?
2. Which prerequisite enables advanced governance features?
3. How do agents built on other frameworks appear in Control Plane?
4. How does Control Plane differ from Agent 365?
5. Name three risk dimensions continuous evaluation can monitor.
6. What does Azure Policy add alongside Control Plane?

<details>
<summary>Answers</summary>

1. Inventory, observability, compliance and security (plus quota and resource management under Manage).
2. Configuring an AI gateway (API Management associated with the Foundry resource).
3. By registering them as custom agents through the AI gateway, so their traffic and telemetry are captured.
4. Control Plane governs AI assets on the Azure/Foundry side (agents, models, tools, quality, cost, guardrails); Agent 365 manages agents as part of the Microsoft 365 workforce (identity, access, lifecycle, usage).
5. Any three of: task adherence, intent resolution, tool call success, groundedness, sensitive data leakage, jailbreak and cross-domain prompt injection exposure.
6. Enforcement of resource configuration (for example disabling local auth, requiring private networking, restricting allowed models and deployment types).

</details>

## References

- [What is Microsoft Foundry Control Plane?](https://learn.microsoft.com/azure/foundry/control-plane/overview)
- [Enable AI gateway in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/configuration/enable-ai-api-management-gateway-portal)
- [Register custom agents](https://learn.microsoft.com/azure/foundry/control-plane/register-custom-agent)
- [Manage agents across the fleet](https://learn.microsoft.com/azure/foundry/control-plane/how-to-manage-agents)
- [Create a guardrail policy](https://learn.microsoft.com/azure/foundry/control-plane/quickstart-create-guardrail-policy)
- [AI Red Teaming Agent](https://learn.microsoft.com/azure/foundry/concepts/ai-red-teaming-agent)
- [Agent 365 integration with Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration)
