# The Microsoft AI Stack

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ● C ● · Status checked: 2026-11

## Why it matters

Clients don't ask "should I use Foundry?". They ask "we have Microsoft 365 Copilot, someone built something in Copilot Studio, the data team uses Fabric and Databricks, and IT wants governance: what goes where?" Microsoft's AI offerings overlap at the edges and are renamed often. Being able to place each product correctly, and explain how they connect through shared identity, data, models and governance, is one of the most valuable skills for architects and consultants.

## Key concepts

### The layers

```text
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Experiences (buy)        Microsoft 365 Copilot · Copilot in Dynamics 365 / Power Platform │
│                          · GitHub Copilot · Security Copilot · Azure Copilot              │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Extend & build low-code  Copilot Studio (agents, topics, actions, publish to M365 Copilot,│
│                          Teams, web) · Agent Builder in M365 Copilot                      │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Build pro-code           Microsoft Foundry: Foundry Models · Foundry Agent Service ·       │
│                          Foundry IQ · Foundry Tools · evaluations · Foundry Control Plane │
│                          + Microsoft Agent Framework (SDK) · Foundry Local (on device)    │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Context ("IQ") layer     Work IQ (M365 Graph) · Foundry IQ (knowledge bases on AI Search) │
│                          · Fabric IQ (semantic models, ontology, data agents) · Web IQ    │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Data & ML                Microsoft Fabric · Azure Databricks · Azure Machine Learning ·   │
│                          Azure databases & storage                                        │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Trust & governance       Microsoft Entra (incl. Agent ID) · Purview · Defender · Content  │
│                          Safety · Agent 365 · API Management AI gateway · Azure Policy    │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Infrastructure           Azure AI infrastructure (GPU VMs, AKS, Container Apps), regions  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### The main products

| Product | What it is | Primary users | Typical use |
|---|---|---|---|
| **Microsoft 365 Copilot** | AI assistant across Word, Excel, PowerPoint, Outlook, Teams and Copilot Chat, grounded in the user's Microsoft 365 data via Microsoft Graph (Work IQ), respecting permissions | Information workers | Productivity: drafting, summarising meetings, finding information |
| **Copilot Studio** | Low-code platform to build agents and extend Microsoft 365 Copilot: knowledge sources, topics, actions (connectors, flows, MCP), autonomous triggers; publish to Teams, M365 Copilot, websites and more | Makers, business technologists, pro developers for extensions | Departmental agents, customer service bots, extending Copilot |
| **Microsoft Foundry** | Azure's platform for building, deploying and governing AI apps and agents: model catalogue and deployments, Agent Service, Foundry IQ, Foundry Tools, evaluations, observability, Control Plane | Developers, AI engineers, architects | Custom AI apps, RAG, agents, AI in products |
| **Microsoft Agent Framework** | Open-source SDK (.NET, Python) for agents and multi-agent workflows; successor to Semantic Kernel and AutoGen | Pro developers | Code-first agents that can run anywhere and deploy to Foundry |
| **Foundry Tools** (formerly Azure AI services) | Prebuilt AI APIs: Speech, Language, Translator, Vision, Content Understanding, Document Intelligence, Content Safety and more | Developers | Specific perception and language tasks |
| **Azure Machine Learning** | End-to-end ML platform: training, MLOps, pipelines, registries, managed endpoints | Data scientists, ML engineers | Custom model training and lifecycle; classic ML |
| **Microsoft Fabric** | SaaS data platform (OneLake, engineering, warehouse, real-time, Power BI) with Copilot and data agents | Data engineers, analysts | The data foundation for AI; data agents as agent knowledge |
| **Azure Databricks** | Spark and AI platform with Unity Catalog and Mosaic AI | Data and ML engineers | Heavy data engineering and ML |
| **Agent 365** | Microsoft 365-side control plane to manage and govern agents (including Foundry and Copilot Studio agents) as part of the workforce | IT admins, security | Agent inventory, access, lifecycle in Microsoft 365 |

Product names and licensing details change frequently. Confirm against Microsoft Learn and the current licensing guides before advising clients.

### How they connect

| Connection | How |
|---|---|
| **Foundry agents → Microsoft 365 Copilot and Teams** | Publish Foundry agents to Microsoft 365 Copilot and Teams |
| **Copilot Studio → Foundry** | Use Foundry models in Copilot Studio; call Foundry agents from Copilot Studio (connected agents); Copilot Studio agents can use Azure AI Search knowledge |
| **Foundry → Fabric** | Fabric data agents as tools in Foundry agents (user identity passthrough); OneLake data indexed into AI Search |
| **Foundry → Microsoft 365 data** | Work IQ / Microsoft Graph connectors and APIs, SharePoint knowledge sources in Foundry IQ (permission-aware) |
| **Azure ML ↔ Foundry** | Shared model catalogue; managed compute endpoints; Azure ML for training and MLOps, Foundry for apps and agents |
| **Identity** | Microsoft Entra ID for users, apps, managed identities, and **Entra Agent ID** for agent identities across the stack |
| **Governance** | Foundry Control Plane (Azure-side fleet governance), Agent 365 (Microsoft 365-side), Purview (data security and compliance for Copilot and agents), Defender for Cloud (AI threat protection) |

### Buy, extend or build

| Option | Choose when | Examples |
|---|---|---|
| **Buy** (Microsoft 365 Copilot, Dynamics 365 Copilot, GitHub Copilot) | The need is general productivity in Microsoft apps | Meeting summaries, drafting, code completion |
| **Extend** (Copilot Studio, agents in Microsoft 365 Copilot) | You need a department-specific agent over business data, mostly low-code, for internal users in Microsoft 365 | HR policy agent in Teams; IT helpdesk agent with ServiceNow connector |
| **Build** (Foundry, Agent Framework) | Custom UX, external customers, complex integration, strict control of models, data, networking and evaluation, or embedding AI in your product | Customer-facing support agent; claims processing; AI features in a SaaS product |

Many organisations use all three: Microsoft 365 Copilot for everyone, Copilot Studio for departmental agents, Foundry for strategic custom solutions, with shared governance.

## How it works on Azure

### Example enterprise landscape

```text
Employees ── Microsoft 365 Copilot ── agents built in Copilot Studio (HR, IT) ──┐
                                   └─ Foundry agent "Claims assistant" (published to Teams/M365) ─┐
Customers ── web/mobile app ── Foundry agent "Customer support" ───────────────────────────────┤
                                                                                               ▼
                         Foundry: models (GPT, Claude, Phi…), Foundry IQ (AI Search), tools (MCP via APIM),
                                  evaluations, tracing (App Insights), Control Plane
                                                                                               ▼
                         Fabric (OneLake, semantic models, data agents) · Dynamics/SAP via APIs
Governance: Entra (users, managed identities, Agent ID) · Purview · Defender · APIM AI gateway · Agent 365
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Copilot Studio vs Foundry | Copilot Studio for low-code, Microsoft 365-centric, maker-led agents with connectors; Foundry for pro-code control, custom models, complex orchestration, external users, rigorous evaluation and networking. They interoperate |
| Foundry vs Azure ML | Foundry for generative AI apps and agents and model consumption; Azure ML for training custom models, classic ML and full MLOps |
| Fabric data agents vs custom SQL tools | Data agents for governed natural-language Q&A over Fabric with user identity; custom tools when you need specific write actions or non-Fabric data |
| Where governance lives | Combine Foundry Control Plane (Azure resources, agents, models) with Agent 365 and Purview (Microsoft 365 and data). Avoid unmanaged "shadow agents" |

## Hands-on

1. Draw your organisation's (or a sample client's) AI landscape on the layer diagram above, listing current products and gaps.
2. Build the same simple FAQ agent twice: once in Copilot Studio with a SharePoint knowledge source, once in Foundry with a knowledge base. Compare effort, control, cost model and governance.
3. Publish the Foundry agent to Microsoft Teams or Microsoft 365 Copilot (if your tenant allows) and test it there.
4. Write a one-page "buy, extend or build" recommendation for three requested use cases.

## Security, cost and reliability notes

- **Security:** every layer uses Microsoft Entra. Align Conditional Access, agent identities and least privilege across Copilot Studio and Foundry agents.
- **Security:** Microsoft 365 Copilot respects existing permissions, so oversharing in SharePoint becomes visible through Copilot. Fix oversharing first (Purview, SharePoint Advanced Management).
- **Cost:** licensing models differ: per-user licences (Microsoft 365 Copilot), capacity or message-based billing (Copilot Studio), consumption (Foundry models, tokens), capacity (Fabric). Model total cost per use case.
- **Reliability:** cross-product solutions depend on several services; design monitoring end to end (Application Insights tracing, Foundry observability).

## Common mistakes

- Positioning Foundry and Copilot Studio as competitors instead of complementary tools.
- Building a custom RAG app for something Microsoft 365 Copilot already does for internal Microsoft 365 content.
- Using outdated names (Azure AI Studio, Azure AI services, Azure OpenAI Service) in client proposals.
- Ignoring governance until there are hundreds of unmanaged agents.
- Recommending Azure ML for a generative AI app that needs no training.

## Check yourself

1. What's the difference between Microsoft 365 Copilot, Copilot Studio and Microsoft Foundry?
2. When would you build in Foundry instead of Copilot Studio?
3. What is the Microsoft Agent Framework and what did it replace?
4. Name the four IQ layers and what each provides.
5. How can a Foundry agent reach users in Microsoft 365?
6. Which products govern agents, and on which side (Azure or Microsoft 365)?

<details>
<summary>Answers</summary>

1. Microsoft 365 Copilot is a ready-made assistant across Microsoft 365 apps; Copilot Studio is a low-code platform to build agents and extend Copilot; Foundry is Azure's pro-code platform to build, deploy and govern custom AI apps and agents.
2. When you need custom UX or external users, complex integration, specific models, network isolation, rigorous evaluation and pro-code control, or AI embedded in your own product.
3. An open-source SDK for building agents and multi-agent workflows in .NET and Python, the successor to Semantic Kernel and AutoGen.
4. Work IQ (how people work, from Microsoft 365), Foundry IQ (documents and knowledge bases), Fabric IQ (business data and semantics), Web IQ (the web).
5. By publishing it to Microsoft Teams and Microsoft 365 Copilot, or by calling it from a Copilot Studio agent.
6. Foundry Control Plane on the Azure side; Agent 365 on the Microsoft 365 side; Microsoft Entra Agent ID for identities; Purview and Defender for data security and threat protection across both.

</details>

## References

- [What is Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/what-is-foundry)
- [Microsoft 365 Copilot overview](https://learn.microsoft.com/copilot/microsoft-365/microsoft-365-copilot-overview)
- [Copilot Studio overview](https://learn.microsoft.com/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [Microsoft IQ](https://learn.microsoft.com/microsoft-iq)
- [Agent 365 integration with Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration)
- [What is Azure Machine Learning?](https://learn.microsoft.com/azure/machine-learning/overview-what-is-azure-machine-learning)
- [Choose a Microsoft AI technology (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/strategy)
