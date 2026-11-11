# Microsoft 365 Copilot Architecture

## Why it matters

For most organisations, the first enterprise AI deployment isn't a custom Foundry agent: it's Microsoft 365 Copilot in Word, Outlook, Teams and the Copilot app. Architects and consultants are asked to explain how it works, what data it can see, whether it's safe to switch on, how licensing works, and how it relates to custom agents built on Azure. The answers depend on understanding the architecture: an orchestration engine that grounds large language models in **Microsoft Graph** and **Work IQ** using the **semantic index**, scoped strictly to each user's existing permissions, protected by Enterprise Data Protection, and extensible with agents. Get this right and you can plan readiness (especially oversharing), set expectations and decide when Copilot is enough and when to build.

## Key concepts

### Components

Microsoft describes Copilot as a processing and orchestration engine coordinating:

| Component | Role |
|---|---|
| **Large language models** | Hosted by Microsoft (with OpenAI and Anthropic models as subprocessors in applicable licensed experiences); a real-time router chooses between quick-response and "think deeper" reasoning models in **Auto** mode, or the user picks via the model selector |
| **Microsoft Graph** | Emails, chats, files, meetings, calendars, people and relationships the user can access |
| **Work IQ** | Intelligence layer letting Copilot and agents reason over organisational data, content and tools; can be turned on or off. A separate usage-based **Work IQ API** serves custom apps and agents |
| **Semantic index** | Lexical and semantic index over Graph data for precise retrieval; honours the user's identity-based access boundary |
| **Microsoft 365 apps** | Word, Excel, PowerPoint, Outlook, Teams, OneNote, Loop and others where Copilot acts in context |
| **Copilot Search** | Enterprise search across Microsoft 365 and connected sources |
| **Agents and connectors** | Extend knowledge and actions (file 02) |
| **Web grounding** | Bing web search queries, controlled by the "Allow web search in Copilot" policy |

### Request flow

```text
User prompt in an app
  └─► Copilot orchestrator
        ├─ pre-processing (responsible AI checks, prompt injection protection)
        ├─ grounding: Graph + semantic index + Work IQ (user's permissions only), web search if allowed, agents if invoked
        ├─ LLM generates response with the grounded context
        ├─ post-processing (harmful content and protected material checks, compliance, labels)
        └─► response with citations; interaction stored for the user (subject to Purview retention)
```

Copilot only presents data each user can already access, using the same access controls as other Microsoft 365 services. It inherits sensitivity labels of referenced content; encrypted content needs VIEW and EXTRACT rights.

### Licences and experiences

| Experience | Grounding | Notes |
|---|---|---|
| **Copilot Chat (Basic)** | Web; organisational content only when uploaded or via pay-as-you-go agents | Included with eligible Microsoft 365 licences; copilot.cloud.microsoft, Copilot app, Edge, Outlook, Teams |
| **Microsoft 365 Copilot (Basic)** | Web; standard access to Copilot in Word, Excel, PowerPoint, OneNote | Fair-use limits |
| **Microsoft 365 Copilot (Premium)** | Web **and** work data via Graph and Work IQ automatically | Add-on licence; agents with work data; advanced admin controls and analytics |
| **Usage-based billing** | Adds **Copilot Cowork** and **Copilot Code** | Consumption billing with admin usage and cost controls |

All three licensed experiences are covered by **Enterprise Data Protection (EDP)** when users sign in with Entra accounts: prompts and responses get the same contractual protections as other Microsoft 365 data. Licensing changes often: confirm current names and entitlements in Microsoft's documentation.

### Data, privacy and residency

| Topic | Commitment |
|---|---|
| Training | Prompts, responses and Graph data aren't used to train foundation LLMs |
| Compliance | GDPR, EU Data Boundary; covered workload in data residency commitments (Product Terms) since 1 March 2024; Advanced Data Residency and Multi-Geo |
| Stored data | Prompts and responses with citations, stored for the user; discoverable with Content search/eDiscovery; Purview retention policies; users can delete history in My Account |
| Protections | Harmful content blocking, protected material detection, prompt injection (jailbreak) blocking |
| Extensibility data | When agents are used, Copilot sends a query to the agent on the user's behalf; admins see each agent's permissions, terms and privacy statement in Integrated apps |

### Admin controls

**Copilot controls** in the Microsoft 365 admin centre: pinning Copilot Chat, image generation, how users create and use agents, web search grounding, removing access. The **AI Administrator** role gives least-privilege management without Global Administrator.

## How it works on Azure

Microsoft 365 Copilot is SaaS: you don't deploy it on Azure. Azure enters the picture when you:

- extend Copilot with **custom engine agents** hosted on Azure (Foundry, Azure Bot Service, App Service) (file 02, file 04);
- build **API plugins** or MCP servers on Azure (Functions, App Service, API Management) that declarative agents call;
- bring external data in with **Copilot connectors**, which may read from Azure data stores;
- govern with **Entra** (Conditional Access, Agent ID), **Purview** (labels, DLP, DSPM, retention) and **Defender** across both Microsoft 365 and Azure.

Readiness checklist:

| Area | Action |
|---|---|
| Permissions | Oversharing assessment (Purview DSPM), SharePoint Advanced Management, Restricted Content Discovery for risky sites ([Topic 23 file 05](../23-responsible-ai-security-and-governance/05-data-protection-for-ai.md)) |
| Labels and DLP | Sensitivity labels, auto-labelling, DLP to exclude highly confidential content from Copilot |
| Network | Allow `*.cloud.microsoft` and other required endpoints |
| Licensing | Decide Basic vs Premium by role; pilot cohort |
| Adoption | Champions, training, scenario library, Copilot Dashboard in Viva Insights |
| Governance | Agent creation and sharing policy; AI Administrator role; retention |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Premium for whom** | Roles that work heavily with organisational content (managers, sales, analysts); Basic or Chat for others; measure before broad rollout. |
| **Web grounding** | Usually allowed; disable for highly regulated roles if policy requires. |
| **Work IQ** | On for Premium users; consider effects on data exposure before enabling broadly. |
| **Remediate vs restrict** | Long-term permission remediation; short-term Restricted Content Discovery. |
| **Copilot vs custom** | Use Copilot for general productivity; build agents only where Copilot lacks the data, actions or control you need (file 05). |

## Hands-on

1. Read your tenant's Copilot controls in the Microsoft 365 admin centre; document current settings.
2. Run a Purview oversharing assessment and identify the five riskiest sites.
3. As a test user, ask Copilot about content in a site you can and can't access; confirm permission trimming and citations.
4. Configure a Purview retention policy for Copilot interactions.
5. Open the Copilot Dashboard and define three adoption metrics for a pilot.

## Security, cost and reliability notes

- **Permissions are the security model**: oversharing becomes over-answering.
- **Prompts are records**: retention, eDiscovery and deletion rights apply.
- **Cost**: per-user licences plus usage-based billing for Cowork, Code and pay-as-you-go agents; monitor in the admin centre.
- **Model changes**: Microsoft updates models; Copilot responses change over time; communicate to users.

## Common mistakes

| Mistake | Fix |
|---|---|
| Switching Copilot on before fixing oversharing | Assess and remediate first |
| Assuming Copilot sees "everything" | It sees only what the user can access |
| Buying Premium for everyone without a pilot | Pilot, measure, expand |
| Ignoring agent governance | Copilot controls and Agent 365 (file 06) |
| Confusing Copilot Chat (Basic) with Premium capabilities | Explain grounding differences clearly |

## Check yourself

1. Name the main components Copilot orchestrates.
2. What is the semantic index, and how does it respect permissions?
3. Compare Copilot Chat (Basic), Microsoft 365 Copilot (Basic) and Premium grounding.
4. What does Enterprise Data Protection provide?
5. Where are Copilot interactions stored, and how are they governed?
6. Which admin controls exist for Copilot?

<details>
<summary>Answers</summary>

1. LLMs, Microsoft Graph, Work IQ, semantic index, Microsoft 365 apps, Copilot Search, agents/connectors and web grounding.
2. A lexical and semantic index over Graph data for precise retrieval that honours the user's identity-based access boundary.
3. Chat (Basic) and M365 Copilot (Basic): web grounding, organisational content only if uploaded or via pay-as-you-go agents (Basic adds in-app access). Premium: web plus Graph and Work IQ automatically.
4. Prompts and responses receive the same contractual protections as other Microsoft 365 data when signed in with Entra accounts.
5. Stored for the user with citations; searchable via Content search/eDiscovery, governed by Purview retention; users can delete their history.
6. Pinning, image generation, agent creation and use, web search grounding, removing access; AI Administrator role for delegated management.

</details>

## References

- [Microsoft Copilot overview](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-architecture)
- [Data, privacy and security for Microsoft Copilot](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-privacy)
- [Microsoft 365 Copilot extensibility](https://learn.microsoft.com/microsoft-365-copilot/extensibility/)
- [Microsoft Purview for AI](https://learn.microsoft.com/purview/ai-microsoft-purview)
- [Microsoft 365 Copilot Dashboard](https://learn.microsoft.com/viva/insights/org-team-insights/copilot-dashboard)
