# Copilot Studio

## Why it matters

Many agents don't need a development team: an HR policy assistant, an IT helpdesk agent that resets passwords through an existing flow, a customer-service agent on the website, a workflow that triages invoices. **Microsoft Copilot Studio** is Microsoft's low-code platform for building such agents and workflows, connecting them to data and 1,000+ Power Platform connectors, and publishing them to Microsoft 365 Copilot, Teams, websites and other channels, with governance from Power Platform and Microsoft 365. Architects need to know what it can do, which **harness** to choose, how it extends with MCP and Foundry, and how it's governed and billed, because it often competes with (or complements) a Foundry build.

## Key concepts

### Building blocks

| Block | What it is |
|---|---|
| **Agent** | AI assistant following instructions, using knowledge and tools; created by describing it in natural language; can be given its own account to work proactively |
| **Workflow** | Automation built in a drag-and-drop designer where each step can reason and act, combining agents, tools and logic |
| **Agent flow** | Established flow format (Power Automate-like) run standalone or attached to an agent as a tool |
| **Topic** | Designed conversation path (standard harness) with triggers, questions, conditions |
| **Knowledge** | SharePoint, OneDrive, Dataverse, websites, uploaded files, connectors, Azure AI Search and others |
| **Tools** | Connectors, agent flows, prompts, REST APIs, MCP servers, computer use, child and connected agents |
| **Triggers** | Event triggers that start autonomous runs (for example, a new email or Dataverse record) |
| **Apps** (Preview) | App creation alongside agents and workflows |

### Harnesses

Everything runs on a **harness**, the runtime between your design and the model, which affects capability and billing:

| | GitHub Copilot harness | Customer Conversation harness | Standard harness |
|---|---|---|---|
| **Best for** | Complex, multi-step business processes | Customer-facing contact-centre scenarios over voice and chat | Rule-based agents and structured conversations |
| **How it works** | Reasons through a goal step by step, chooses tools, retries and finds alternative paths | Primary agent plus specialised sub-agents; escalation to a human | Follows the topics and rules you define (generative or classic orchestration) |
| **Files** | Creates, edits and reasons over Word, Excel, PowerPoint, PDF | Not a focus | Not a focus |

The GitHub Copilot harness is a Copilot Studio framework sharing technology with GitHub Copilot; customer data isn't sent to the GitHub Copilot service.

### Orchestration on the standard harness

| Behaviour | Generative orchestration (default) | Classic orchestration |
|---|---|---|
| Topic selection | By description | By trigger phrases |
| Tools | Agent chooses by name and description | Only called from within topics |
| Knowledge | Proactively searched | Fallback when no topic matches |
| Combining | Multiple topics, tools, knowledge in one turn | Single topic |
| Missing inputs | Agent generates questions | Question nodes required |
| Child/connected agents | Selected by description | Not applicable |

MCP requires generative orchestration. Write precise names and descriptions for tools, topics and agents, since the orchestrator selects by them.

### MCP in Copilot Studio

Connect MCP servers through the onboarding wizard; tools and resources (prompts not yet) become available automatically and update dynamically as the server changes. You're responsible for third-party MCP servers you connect. MCP connectors can be published for use across tenants.

### Autonomous agents

Agents with **event triggers** run without a user prompt: for example, when an email arrives, classify it, look up the customer in Dataverse, draft a reply and create a case. Triggers run under the author's or a configured connection's credentials, so review permissions carefully. Agents can also receive their own account (agent user identity) for proactive work.

### Lifecycle and operations

| Area | Capability |
|---|---|
| **Analytics** | Session outcomes (resolved, escalated, abandoned), CSAT, themes, custom metrics; 28-day transcripts, up to 360 days of monitor data |
| **Evaluations** | Test sets and a shared grader library before and after publishing |
| **ALM** | Solutions, environments (dev/test/prod), pipelines |
| **Administration** | Agent inventory, role-based access, cost management in the Power Platform admin centre |
| **Governance** | Power Platform DLP policies on connectors (including knowledge and MCP), environment strategy, managed environments, sharing limits |

### Licensing

Billing depends on harness: usage-based billing (Copilot credits / consumption) for the GitHub Copilot harness, and capacity or pay-as-you-go licensing for standard harness agents. Microsoft 365 Copilot (Premium) users get included use of agents in Microsoft 365 experiences. Licensing changes frequently; check the current licensing guide.

## How it works on Azure

Copilot Studio is SaaS on the Power Platform, but Azure often supplies the heavy lifting:

| Need | Azure integration |
|---|---|
| Enterprise search knowledge | **Azure AI Search** index as a knowledge source |
| Custom APIs and MCP servers | Azure Functions / Container Apps behind **API Management**, connected as REST or MCP tools |
| Custom models for prompts | **Foundry models** in the Prompt tool (file 04) |
| Complex agents | **Foundry agents** as connected agents (file 04) |
| Telemetry | Export to **Application Insights** |
| Network isolation | Power Platform **VNet support** (subnet delegation) for connectors reaching private Azure resources |
| Identity | Entra authentication for users and connections; Entra Agent ID for agent identities |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Harness** | Standard for predictable FAQ and guided flows; Customer Conversation for contact-centre agents; GitHub Copilot harness for complex multi-step processes and files. |
| **Generative vs classic** | Generative by default; classic or topics where you need deterministic, auditable paths. |
| **Knowledge source** | SharePoint for documents users can access; Azure AI Search for large, curated or multi-source corpora. |
| **Autonomous triggers** | Start with human review of outputs; limit connection permissions. |
| **Environment strategy** | Default environment for personal productivity only; managed dev/test/prod environments for shared agents. |
| **When to leave low-code** | Need custom models, complex evaluation, high scale or code-level control → Foundry (file 05). |

## Hands-on

1. Create an IT helpdesk agent by describing it; add a SharePoint knowledge source and a topic for password resets that calls an agent flow.
2. Add an MCP server tool (Azure Functions behind APIM) and verify generative orchestration chooses it.
3. Add an event trigger that processes new emails to a shared mailbox and drafts replies for review.
4. Create a test set and run an evaluation; review analytics after 20 sessions.
5. Apply a Power Platform DLP policy blocking a consumer connector and see its effect on the agent.

## Security, cost and reliability notes

- **Connections**: agents act through connections; avoid makers' personal credentials for shared agents; use service connections and least privilege.
- **DLP**: classify connectors into business, non-business and blocked; include MCP and HTTP connectors.
- **Prompt injection**: knowledge and email triggers ingest untrusted content; restrict actions and require approval for sensitive ones.
- **Cost**: monitor consumption per environment; set capacity alerts.
- **Sharing**: limit who can share agents org-wide; require admin publishing (file 06).

## Common mistakes

| Mistake | Fix |
|---|---|
| Building everything in the default environment | Managed environments and ALM |
| Vague tool and topic descriptions | Precise metadata for the orchestrator |
| Autonomous agent with a maker's admin credentials | Dedicated, least-privilege connections |
| No evaluation before publishing | Test sets and graders |
| Ignoring harness choice | Choose by scenario and billing |

## Check yourself

1. Name Copilot Studio's main building blocks.
2. Compare the three harnesses.
3. How does generative orchestration differ from classic?
4. What does MCP support in Copilot Studio include, and what's required?
5. What makes an agent autonomous, and what's the main risk?
6. How is Copilot Studio governed?

<details>
<summary>Answers</summary>

1. Agents, workflows, agent flows, topics, knowledge, tools, triggers (and apps in Preview).
2. GitHub Copilot harness: reasoning, multi-step processes, files. Customer Conversation: customer-facing voice/chat with sub-agents and human escalation. Standard: rule-based topics with generative or classic orchestration.
3. Generative selects topics, tools, knowledge and agents by description and can combine them and ask for missing inputs; classic uses trigger phrases, one topic and explicit tool calls.
4. MCP tools and resources with dynamic updates; generative orchestration must be on.
5. Event triggers that start runs without a user; risk is untrusted input driving actions under broad connection permissions.
6. Power Platform environments and managed environments, DLP policies, solutions/ALM, admin centre inventory, sharing controls, plus Microsoft 365 and Agent 365 governance.

</details>

## References

- [Copilot Studio overview](https://learn.microsoft.com/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
- [Harnesses in Copilot Studio](https://learn.microsoft.com/microsoft-copilot-studio/harnesses-overview)
- [Orchestrate agent behaviour with generative AI](https://learn.microsoft.com/microsoft-copilot-studio/advanced-generative-actions)
- [Extend your agent with MCP](https://learn.microsoft.com/microsoft-copilot-studio/agent-extend-action-mcp)
- [Triggers in Copilot Studio](https://learn.microsoft.com/microsoft-copilot-studio/authoring-triggers-about)
- [Copilot Studio analytics](https://learn.microsoft.com/microsoft-copilot-studio/analytics-overview)
