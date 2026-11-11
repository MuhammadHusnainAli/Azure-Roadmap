# Agent Governance at Scale

## Why it matters

The first agent is easy to govern: you know who built it, what it does and who uses it. The five-hundredth is not. Agents are being built in Foundry, Copilot Studio, Microsoft 365 Agent Builder, GitHub, Power Platform, third-party SaaS and custom code on any cloud; users share them with colleagues; vendors ship them inside products. Microsoft calls the result **agent sprawl**: agents multiply with no clear oversight, security teams lack a complete picture, and nobody can say which agents can access what. Microsoft's answer is three products working together: **Microsoft Entra Agent ID** (identity), **Microsoft Foundry Control Plane** (the build-and-operate fleet view for AI assets in Azure) and **Microsoft Agent 365** (the enterprise control plane for agents across Microsoft 365 and beyond), with Purview and Defender supplying data and threat signals. This file shows how they fit and how to run them as one governance system.

## Key concepts

### The three pillars

| | Microsoft Entra Agent ID | Foundry Control Plane | Microsoft Agent 365 |
|---|---|---|---|
| **Question** | Who is this agent and what may it access? | How healthy, safe, compliant and costly are my Azure AI agents, models and tools? | Which agents exist across the enterprise, who owns them, and are they observed, governed and secured? |
| **Audience** | Identity admins | AI platform teams, builders, AI admins | Microsoft 365 admins, security, compliance |
| **Scope** | Agent identity blueprints, agent identities, agent users | Foundry resources across a subscription; custom agents and MCP tools registered via AI gateway | Microsoft, partner and organisation agents; Foundry agents published to it |
| **Key capabilities** | Conditional Access, ID Protection, access reviews, lifecycle, sponsorship and ownership | Assets inventory, fleet health, evaluations, red teaming, guardrail policies, bulk remediation, cost and quota | Registry (Microsoft 365 admin centre), agent map, access control, Purview and Defender integration, lifecycle |
| **Status** | GA (some features Preview) | GA with Preview features | **GA since 1 May 2026** (commercial, per-user licensing; E5 recommended) |
| **More detail** | [Topic 02 file 10](../02-identity-and-access/10-microsoft-entra-agent-id.md) | [Topic 14 file 09](../14-microsoft-foundry-platform/09-foundry-control-plane.md) | [Topic 24](../24-copilot-and-low-code-agents/README.md) file 06 |

Supporting signals: **Purview** (DSPM, DLP, audit for agent interactions, file 05), **Defender** (AI threat protection, AI agent inventory and posture, which since 1 July 2026 requires Agent 365 licensing, file 06), **Azure Policy** (resource configuration), **API Management AI gateway** (runtime traffic for models, agents and MCP tools).

### Agent 365 pillars

| Pillar | Meaning |
|---|---|
| **Identity** | First-class agent identities with accountability via Entra Agent ID |
| **Observability** | See what agents do, which systems and data they access, and how behaviour evolves |
| **Governance** | Central policy controls on data access and sensitive actions |
| **Security** | Detect anomalous behaviour, limit high-risk actions, support response |
| **Lifecycle** | From creation to deployment to retirement; versions and updates |

The **registry** lists agents by type: Microsoft agents, external partner-built agents, published by your org (line-of-business agents) and shared by creator. It highlights **agents without owners**. One Copilot Studio agent can appear as several entries (shared by creator and published by org) by design; admin actions (delete, remove, block) differ by entry type.

### Controlling agent sprawl: the lifecycle

| Stage | Control | Tool |
|---|---|---|
| **Create** | Approved platforms and templates; environment strategy for makers; intake for high-risk agents | Power Platform environments and DLP, Foundry projects, governance intake (file 09) |
| **Identify** | Every agent gets a distinct identity and an owner/sponsor | Entra Agent ID |
| **Register** | Every agent appears in an inventory | Agent 365 registry; Foundry Control Plane assets; custom agents via AI gateway |
| **Authorise** | Least-privilege access; Conditional Access; approval for sensitive actions | Entra, RBAC, tool approvals (file 04) |
| **Publish** | Admin approval before org-wide availability | Microsoft 365 admin centre, Foundry publishing |
| **Observe** | Usage, behaviour, quality, cost | Agent 365 observability, Foundry Control Plane, Application Insights |
| **Protect** | Data and threat controls | Purview, Defender, guardrails |
| **Review** | Periodic access reviews and recertification; orphaned-agent detection | Entra access reviews, registry "agents without owners" |
| **Retire** | Block, remove, delete; disable identities; clean up connections and data | Admin centre, Entra, Foundry |

### Division of responsibility

```text
                Agent 365 (enterprise registry, M365 governance, security integration)
                     ▲ publish / register                     ▲ signals
Foundry Control Plane (Azure fleet: quality, safety, cost, policy) ── Purview / Defender
                     ▲ identity
               Entra Agent ID (identity, Conditional Access, lifecycle)
```

Foundry agents can be **published to Agent 365** for central observability; Control Plane remains the place to evaluate, red team and remediate them. Copilot Studio and Microsoft 365 agents are governed primarily through the Microsoft 365 and Power Platform admin centres and Agent 365.

## How it works on Azure

A practical rollout sequence:

1. **Discover**: Agent 365 registry, Foundry Control Plane assets, Defender AI inventory, Purview DSPM apps and agents, Power Platform inventory, Resource Graph. Reconcile into one list.
2. **Assign owners**: fix "agents without owners"; require a sponsor in Entra for every agent identity.
3. **Baseline policy**: Azure Policy for AI resources; Control Plane guardrail policies with bulk remediation; Power Platform DLP for maker connectors.
4. **Identity hygiene**: move agents to per-agent Entra Agent IDs; remove shared secrets; Conditional Access for agent identities; quarterly access reviews.
5. **Gateway**: route models, custom agents and MCP tools through the API Management AI gateway so they can be registered, limited and logged.
6. **Signals**: enable Purview for Foundry and Defender for AI; integrate alerts into the SOC.
7. **Operate**: monthly fleet review (new, ownerless, non-compliant, costly, unused agents); retire what isn't used.

Example KPI set for an agent governance dashboard:

| KPI | Target |
|---|---|
| Agents with an owner and sponsor | 100% |
| Agents registered in the registry/inventory | 100% |
| Agents with per-agent identity (no shared secrets) | > 95% |
| Production agents with explicit guardrail and evaluation evidence | 100% |
| Agents unused for 90 days | Retired or justified |
| High-severity AI alerts with response within SLA | 100% |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Licensing** | Agent 365 is per-user and works best with E5; model the cost against the risk of unmanaged agents and the Defender agent posture dependency. |
| **Where to govern custom agents** | Register through the AI gateway in Control Plane; publish to Agent 365 when they serve Microsoft 365 users. |
| **Maker freedom vs control** | Personal/shared agents allowed in sandbox environments; org-wide publishing requires approval. |
| **Ownership model** | Business owner plus technical owner; sponsor on the identity; automatic escalation when an owner leaves. |
| **Blocking vs deleting** | Block first (reversible) during investigations; delete after retention requirements are met. |
| **Multi-cloud agents** | Gateway registration and Entra identities give partial coverage; document gaps. |

## Hands-on

1. In the Microsoft 365 admin centre, open **Agents → All agents → Registry**; review agent types and agents without owners.
2. In Foundry, open **Operate** and review Assets and Compliance; apply a guardrail policy and remediate one non-compliant agent.
3. Register a custom agent running outside Foundry through the AI gateway.
4. In Entra, review agent identities, add sponsors and create an access review.
5. Build the KPI table above from real data and present a sprawl-reduction plan.

## Security, cost and reliability notes

- **Orphaned agents** keep their permissions; ownerless and unused agents are high-value targets.
- **Shared secrets** in agents defeat identity governance; prioritise migration to agent identities.
- **Cost**: unused agents still incur hosting, indexes and monitoring costs; retirement saves money.
- **Coverage**: no single tool sees every agent; reconcile multiple inventories.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating Agent 365 and Foundry Control Plane as alternatives | Use both for their scopes, joined by Entra identity |
| Inventory without owners | Mandatory owner and sponsor |
| Letting any maker publish org-wide | Admin approval for publishing |
| Ignoring agents built outside Microsoft platforms | Gateway registration and identity |
| No retirement process | Usage-based review and blocking |

## Check yourself

1. Define agent sprawl and its consequences.
2. Compare Entra Agent ID, Foundry Control Plane and Agent 365.
3. What are Agent 365's five pillars?
4. What agent types appear in the Agent 365 registry?
5. How are custom agents running outside Foundry brought under Control Plane governance?
6. List the agent governance lifecycle stages.

<details>
<summary>Answers</summary>

1. Uncontrolled growth of agents across platforms without oversight, leading to unknown access, security gaps, compliance failures and wasted cost.
2. Entra Agent ID: identities, access and lifecycle. Foundry Control Plane: Azure fleet view of agents, models and tools for quality, safety, compliance and cost. Agent 365: enterprise registry and control plane across Microsoft 365, partner and org agents with Purview and Defender integration.
3. Identity, observability, governance, security, lifecycle management.
4. Microsoft agents, external partner-built agents, published by your org, shared by creator.
5. By registering them (and their MCP tools) through the AI gateway in API Management.
6. Create, identify, register, authorise, publish, observe, protect, review, retire.

</details>

## References

- [Overview of Microsoft Agent 365](https://learn.microsoft.com/microsoft-agent-365/overview)
- [Why does an enterprise need Agent 365?](https://learn.microsoft.com/microsoft-agent-365/guidance/why-agent-365-for-enterprise)
- [Agent registry in the Microsoft 365 admin centre](https://learn.microsoft.com/microsoft-agent-365/admin/agent-registry-overview)
- [What is Foundry Control Plane?](https://learn.microsoft.com/azure/foundry/control-plane/overview)
- [What is Microsoft Entra Agent ID?](https://learn.microsoft.com/entra/agent-id/identity-platform/what-is-agent-id)
- [Govern and secure AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)
