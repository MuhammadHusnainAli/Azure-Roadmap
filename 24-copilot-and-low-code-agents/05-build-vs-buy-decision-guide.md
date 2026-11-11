# Build vs Buy Decision Guide

## Why it matters

"Should we use Copilot, build in Copilot Studio, build in Foundry, or write it ourselves?" is the most common question architects and consultants get about AI agents. The wrong answer is expensive in both directions: building custom what Microsoft 365 Copilot already does wastes months, and forcing a complex, regulated, high-scale process into a low-code tool creates fragile solutions that can't be evaluated or controlled. Microsoft's Cloud Adoption Framework (CAF) gives a clear starting principle: **use the most managed option that meets the requirement**, moving from SaaS (buy) to PaaS (build on a platform) to infrastructure only when needed. This file turns that into a repeatable decision process.

## Key concepts

### The spectrum

| Option | Model | Who builds | Control | Time to value |
|---|---|---|---|---|
| **Microsoft 365 Copilot** (and in-product Copilots) | Buy (SaaS) | Nobody; configure and adopt | Lowest | Days |
| **Role-aligned and prebuilt agents** (Security Copilot, Dynamics 365, Researcher, partner agents) | Buy | Configure | Low | Days–weeks |
| **Declarative agents / Agent Builder** | Extend SaaS | Makers, developers | Low–medium | Days |
| **Copilot Studio** | Build on SaaS (low-code) | Makers, fusion teams | Medium | Weeks |
| **Microsoft Foundry** (Agent Service, models, tools) | Build on PaaS | Developers, AI engineers | High | Weeks–months |
| **Custom code on Azure infrastructure** (AKS, VMs, self-hosted models) | Build on IaaS | Engineering teams | Highest | Months |

### CAF's routing logic

- **Individual productivity** → Microsoft 365 Copilot in Microsoft 365 apps.
- **Product-specific help** → in-product Copilots (Azure, GitHub, Fabric, Dynamics 365, Power Platform).
- **Role-specific work** → role-aligned Copilots (security, sales, service, finance).
- **General use** → Microsoft Copilot.
- **Custom domain skills in Copilot** → Microsoft 365 Copilot extensibility.
- **Automating business functions** → Copilot Studio (SaaS with integrated pricing) or Foundry (development platform with APIs).
- **Prebuilt non-generative AI or search** → Foundry Tools and Azure AI Search.
- **Training your own ML models** → Azure Machine Learning.
- **Azure infrastructure** shouldn't be the default; use it only when you need control over models or runtimes that managed platforms can't provide.

### Decision criteria

| Criterion | Pushes towards buy/low-code | Pushes towards Foundry/custom |
|---|---|---|
| **Users and channel** | Microsoft 365 users in Teams/Outlook | External customers, custom apps, APIs, other clouds |
| **Data** | Microsoft 365 content, Dataverse, standard connectors | Large proprietary corpora, complex RAG, real-time systems |
| **Logic** | Guided conversations, approvals, standard workflows | Multi-agent orchestration, code execution, complex reasoning |
| **Models** | Managed models suffice | Specific, fine-tuned, open or multiple models; routing; provisioned throughput |
| **Evaluation and assurance** | Test sets and analytics suffice | Rigorous evaluation, CI gates, red teaming, regulatory evidence |
| **Scale and latency** | Departmental | High volume, strict SLAs |
| **Skills available** | Makers and Power Platform CoE | AI engineering team |
| **Cost model** | Per-user licences, consumption credits | Azure consumption, engineering cost |
| **Differentiation** | Commodity capability | Competitive differentiator, IP |

### Total cost of ownership

| Cost | Buy | Low-code build | Pro-code build |
|---|---|---|---|
| Licences | Per user | Per user/consumption | Minimal |
| Consumption | Included or usage-based | Credits | Models, search, compute, storage |
| Build | None | Maker time | Engineering time |
| Run | Admin and adoption | CoE, ALM | Platform, SRE, evaluation, security |
| Change | Vendor roadmap | Maker changes | Full control, full responsibility |

## How it works on Azure

A decision flow you can run in a workshop:

```text
1. Does an existing Microsoft or partner Copilot/agent meet ≥80% of the need?   ── yes → Buy, configure, adopt
2. Is it a Microsoft 365 user scenario on Copilot's engine (knowledge + some actions)? ── yes → Declarative agent (+ plugins/connectors)
3. Can a maker team build it with connectors, topics, flows; is evaluation by test sets enough? ── yes → Copilot Studio
4. Need custom models, complex orchestration, external channels, rigorous evaluation, scale? ── yes → Foundry (optionally fronted by Copilot Studio)
5. Need runtime/model control managed platforms can't provide (air-gapped, custom serving)? ── yes → Azure infrastructure
```

Hybrid is common: Copilot Studio front end with Foundry agents behind it (file 04); Foundry agents published into Microsoft 365 Copilot (file 02).

Example outcomes:

| Scenario | Recommendation |
|---|---|
| Summarise meetings and draft emails | Microsoft 365 Copilot |
| HR policy Q&A for employees | Declarative agent on SharePoint, or Copilot Studio if workflows needed |
| IT helpdesk with ticket creation and password resets | Copilot Studio with ITSM connector |
| Customer-facing claims assistant handling documents, payments and regulated decisions | Foundry agents with evaluation and guardrails; Copilot Studio or custom web front end |
| SOC investigation assistance | Microsoft Security Copilot (buy), extended with custom agents |
| Contract analysis at scale with a fine-tuned model | Foundry |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Default posture** | Buy before build; low-code before pro-code; PaaS before IaaS. |
| **Exit criteria** | Define when a low-code solution must move to pro-code (volume, complexity, assurance needs). |
| **Avoid duplication** | Don't build a custom chatbot that duplicates Microsoft 365 Copilot for internal knowledge. |
| **Governance parity** | Whatever is chosen gets the same intake, identity, data and security controls (Topic 23). |
| **Vendor lock-in** | Use open protocols (MCP, A2A, OpenTelemetry) and portable components where differentiation matters. |
| **Pilot fast** | CAF recommends rapid pilots in Copilot Studio or Foundry on the hardest steps before investing. |

## Hands-on

1. Take five use cases from your organisation; run each through the decision flow; record the answer and rationale.
2. Build a TCO comparison for one use case across Copilot Studio and Foundry over three years.
3. Write exit criteria for a Copilot Studio agent that might outgrow low-code.
4. Present a one-slide recommendation with the decision criteria table filled in.

## Security, cost and reliability notes

- **Buy doesn't mean no responsibility**: data readiness, permissions, identity and governance are still yours.
- **Low-code sprawl**: easy building leads to many agents; apply environment and publishing governance.
- **Pro-code operational load**: evaluation, monitoring, security and on-call are ongoing costs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting with "let's build a chatbot on Azure OpenAI" | Start from the use case and buy-first principle |
| Choosing by team preference | Use explicit criteria |
| Ignoring licensing in TCO | Include per-user and consumption costs |
| Treating the decision as permanent | Revisit as products and needs evolve |
| One tool for everything | Hybrid architectures |

## Check yourself

1. State CAF's guiding principle for choosing AI options.
2. Map individual productivity, role-specific work and business-process automation to Microsoft offerings.
3. List five criteria that push towards Foundry.
4. When is Azure infrastructure appropriate?
5. What is a common hybrid pattern?
6. What costs belong in a TCO comparison?

<details>
<summary>Answers</summary>

1. Use the most managed option that meets requirements: SaaS before PaaS before infrastructure.
2. Individual productivity: Microsoft 365 Copilot; role-specific: role-aligned Copilots; business automation: Copilot Studio or Foundry.
3. Any five: external channels, large proprietary data/complex RAG, multi-agent or code execution, custom or multiple models, rigorous evaluation and assurance, high scale/latency SLAs, differentiation.
4. When you need control over models or runtimes that managed platforms can't provide.
5. Copilot Studio front end calling Foundry agents, or Foundry agents published into Microsoft 365 Copilot.
6. Licences, consumption, build effort, run/operate costs (platform, evaluation, security), and change costs.

</details>

## References

- [AI strategy (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/strategy)
- [Business plan for AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [Agents for Microsoft 365 Copilot](https://learn.microsoft.com/microsoft-365-copilot/extensibility/agents-overview)
- [Copilot Studio overview](https://learn.microsoft.com/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
