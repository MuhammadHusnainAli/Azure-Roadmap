# AB-100 Agentic AI Business Solutions Architect

## Why it matters

AB-100 (Agentic AI Business Solutions Architect) leads to **Microsoft Certified: Agentic AI Business Solutions Architect Expert**. It targets architects who design AI-driven business solutions across Microsoft Foundry, Copilot Studio, Microsoft 365 Copilot, Power Platform and Dynamics 365. The exam centres on agentic-first and multi-agent design, open protocols (MCP and A2A), and responsible AI and governance. It's the architect-level credential closest to what the **AI Solution Architect** and **Azure Consultant** tracks do day to day: plan, design and deploy AI solutions that deliver measurable business outcomes.

## Key concepts

### Skills measured (study guide, as of 16 November 2026)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Plan AI-powered business solutions** | 35–40% | Analyse requirements; design overall AI strategy; evaluate costs and benefits |
| **Design AI-powered business solutions** | 30–35% | Design AI and agents; design extensibility; orchestrate configuration of prebuilt agents and apps |
| **Deploy AI-powered business solutions** | 20–25% | Analyse, monitor and tune; manage testing; design ALM; design responsible AI, security, governance, risk and compliance |

The 16 November 2026 update increased the weight of the plan and design domains and decreased the deploy domain. If you studied the earlier version, check the change log.

### Prerequisites

To earn the certification, you pass AB-100 **and** hold at least one prerequisite Associate certification. The list includes:

- AI-103 Azure AI Apps and Agents Developer.
- AI-300 Machine Learning Operations Engineer.
- AB-410 Intelligent Applications Builder.
- AB-620 AI Agent Builder.
- AB-210 Dynamics 365 Sales AI Consultant.
- AB-250 Dynamics 365 Contact Center AI Engineer.
- Several Dynamics 365 and Power Platform Associates.

Check the certification page for the current list.

### Audience profile highlights

- Agentic-first and multi-agent orchestrated solution design.
- Knowledge of Dynamics 365, Power Platform, Copilot Studio, Foundry Tools and Foundry Models.
- Open standards: **Agent2Agent (A2A)** and **Model Context Protocol (MCP)**.
- Responsible AI aligned with the Microsoft Responsible AI Standard.
- Securing models and data workflows: data residency, access control and safeguarding model tuning.

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| Agents, multi-agent design, tools | [18 · Agents with Foundry Agent Service](../18-agents-with-foundry-agent-service/README.md), [19 · Agent Frameworks and Protocols](../19-agent-frameworks-and-protocols/README.md) |
| Copilot, Copilot Studio, prebuilt agents | [24 · Copilot and Low-Code Agents](../24-copilot-and-low-code-agents/README.md) |
| Architecture, patterns, operating model | [34 · AI Solution Architecture](../34-ai-solution-architecture/README.md) |
| Requirements, strategy, business case, adoption | [35 · Consulting Practice](../35-consulting-practice/README.md) |
| Testing, monitoring, ALM | [22](../22-evaluation-observability-and-genaiops/README.md), [27](../27-devops-and-platform-engineering/README.md) |
| Responsible AI, security, governance | [23 · Responsible AI, Security and Governance](../23-responsible-ai-security-and-governance/README.md) |

## How it works on Azure

### Eight-week study plan (after one prerequisite Associate)

| Week | Focus | Practice |
|---|---|---|
| 1 | Requirements and AI strategy | Topic 35 files 02–04; build a use-case portfolio |
| 2 | Cost-benefit analysis | Business case with licensing (topic 35 file 10) and token cost scenarios |
| 3 | Agent design | Choose between Copilot Studio, Microsoft 365 Copilot extensibility and Foundry agents for five scenarios |
| 4 | Multi-agent and protocols | Design a handoff or group-chat pattern; expose a tool over MCP; outline A2A interoperability |
| 5 | Extensibility and prebuilt agents | Dynamics 365 and Power Platform agent configuration; connectors |
| 6 | Testing and monitoring | Evaluation strategy, agent analytics, tuning loop |
| 7 | ALM and governance | Environments, solutions, pipelines; Agent 365, Purview and Defender controls |
| 8 | Revision | Practice assessment; [capstone 10 client engagement simulation](../37-capstone-projects/10-client-engagement-simulation.md) |

### Choosing the build surface (a typical exam theme)

| Requirement | Likely choice |
|---|---|
| Business users build and maintain the agent; Microsoft 365 or Dataverse data | Copilot Studio |
| Extend Microsoft 365 Copilot for knowledge workers | Declarative agent or Copilot connector |
| Custom orchestration, code-first tools, model choice, strict evaluation | Foundry Agent Service or Microsoft Agent Framework |
| CRM or contact-centre process | Dynamics 365 agents, extended with Copilot Studio |
| Cross-vendor agent interoperability | A2A; tools exposed via MCP |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prerequisite choice** | Azure-focused architects usually take AI-103; business-apps architects take AB-410 or a Dynamics 365 Associate. |
| **AB-100 vs AZ-305** | They complement each other: AB-100 for business AI design, AZ-305 for the Azure platform. |
| **Depth in Dynamics 365** | You need product breadth, not functional-consultant depth. |

## Hands-on

1. Write an AI strategy and prioritised use-case roadmap for a sample company.
2. Design one solution three ways (Copilot Studio, Microsoft 365 Copilot extensibility, Foundry) and justify the choice.
3. Draft an ALM design covering environments, solutions and promotion.
4. Write a responsible AI and governance plan for a multi-agent solution.
5. Take the practice assessment on AI Skills Navigator.

## Security, cost and reliability notes

- Expect questions on data residency, least-privilege agent identities and governance of agent sprawl.
- Cost-benefit analysis includes licensing (Copilot, Copilot Studio, Agent 365) and consumption.
- Testing and monitoring are part of deployment design, not afterthoughts.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating AB-100 as an Azure-only exam | Study Copilot Studio, Microsoft 365 Copilot, Power Platform and Dynamics 365 too |
| Ignoring business planning | Planning is the largest domain |
| Forgetting the prerequisite | Plan your Associate certification first |
| Studying the pre-November 2026 outline | Use the current study guide |

## Check yourself

1. What are AB-100's three domains, and which is largest?
2. What must you hold in addition to passing AB-100?
3. Which open protocols does the audience profile name?
4. When would you choose Copilot Studio over Foundry Agent Service?
5. Which roadmap topics map to AB-100?

<details>
<summary>Answers</summary>

1. Plan (35–40%, largest), Design (30–35%) and Deploy (20–25%) AI-powered business solutions.
2. At least one prerequisite Associate certification, such as AI-103, AI-300, AB-410 or AB-620.
3. Agent2Agent (A2A) and Model Context Protocol (MCP).
4. When business users build and maintain the agent, it uses low-code connectors, and its data is mainly in Microsoft 365 or Dataverse.
5. Topics 18, 24, 34 and 35, supported by 19, 22, 23 and 27.

</details>

## References

- [Exam AB-100](https://learn.microsoft.com/credentials/certifications/exams/ab-100/)
- [AB-100 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ab-100)
- [Copilot Studio overview](https://learn.microsoft.com/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
- [Microsoft Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/overview)
