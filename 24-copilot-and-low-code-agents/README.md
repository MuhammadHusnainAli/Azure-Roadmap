# 24 · Copilot, Low-Code Agents and Azure AI Assistants

> Tracks: E ○ A ● C ● · Part D · AI on Azure

Not every AI solution is built from scratch on Azure. This topic covers Microsoft's "buy and extend" layer and how it connects to Azure: the Microsoft 365 Copilot architecture (Graph, Work IQ, semantic index, licensing and data protection); extending Copilot with connectors, declarative agents, MCP and API plugins and custom engine agents; Copilot Studio with its harnesses, orchestration, MCP and autonomous triggers; fusion patterns combining Copilot Studio and Foundry; a build-vs-buy decision guide based on the Cloud Adoption Framework; Agent 365 as the enterprise control plane for agents; GitHub Copilot for Azure workloads and app modernisation; Azure Copilot and its agents; the Azure SRE Agent; Microsoft Security Copilot; and AI in Power Platform and Dynamics 365.

## What you will be able to do

- Explain how Microsoft 365 Copilot grounds answers and enforces permissions, and plan readiness.
- Choose between connectors, declarative agents, plugins and custom engine agents.
- Build and govern Copilot Studio agents and pick the right harness.
- Combine Copilot Studio and Foundry with BYO models, connected agents and shared MCP tools.
- Run a defensible build-vs-buy decision with TCO.
- Govern agents with Agent 365's registry, approvals and SDK.
- Use GitHub Copilot, Copilot for Azure and modernisation safely in Azure delivery.
- Use Azure Copilot and its agents for operations, and the SRE Agent for automated incident response.
- Position Security Copilot and Power Platform/Dynamics 365 AI within an enterprise AI portfolio.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) (AI patterns), [Topic 18](../18-agents-with-foundry-agent-service/README.md) (Foundry agents) and [Topic 23](../23-responsible-ai-security-and-governance/README.md) (governance and data protection). Familiarity with Microsoft 365 administration helps.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Microsoft 365 Copilot Architecture](01-microsoft-365-copilot-architecture.md) | 40 min | Readiness and permission trimming |
| 02 | [Extending Microsoft 365 Copilot](02-extending-microsoft-365-copilot.md) | 50 min | Declarative agent with MCP plugin |
| 03 | [Copilot Studio](03-copilot-studio.md) | 50 min | Helpdesk agent with trigger |
| 04 | [Copilot Studio with Foundry](04-copilot-studio-with-foundry.md) | 45 min | BYO model, connected Foundry agent |
| 05 | [Build vs Buy Decision Guide](05-build-vs-buy-decision-guide.md) | 35 min | Decision workshop, TCO |
| 06 | [Agent 365](06-agent-365.md) | 40 min | Registry, approvals, SDK sample |
| 07 | [GitHub Copilot for Azure Workloads](07-github-copilot-for-azure-workloads.md) | 45 min | Instructions, cloud agent, modernisation |
| 08 | [Azure Copilot](08-azure-copilot.md) | 30 min | Queries, troubleshooting agent |
| 09 | [Azure SRE Agent](09-azure-sre-agent.md) | 40 min | Incident investigation in Review mode |
| 10 | [Microsoft Security Copilot](10-microsoft-security-copilot.md) | 35 min | Incident and KQL with promptbooks |
| 11 | [Power Platform and Dynamics 365 AI](11-power-platform-and-dynamics-365-ai.md) | 40 min | AI Builder flow with Foundry prompt |

**AI engineers:** 02, 04, 07 and 09 in depth. **Architects:** all of it, especially 01, 02, 05 and 06. **Consultants:** 01, 03, 05, 06, 10 and 11.

## Completion checklist

- [ ] I can explain Copilot's grounding, permission model, licences and data protection.
- [ ] I have built a declarative agent with an MCP or API plugin on Azure.
- [ ] I have built a Copilot Studio agent with knowledge, tools and a trigger, and chosen a harness.
- [ ] I have connected Copilot Studio to a Foundry model or agent.
- [ ] I have run a build-vs-buy decision with TCO for a real use case.
- [ ] I can use the Agent 365 registry and approval flow and explain the SDK.
- [ ] I have configured GitHub Copilot instructions and reviewed a cloud agent pull request.
- [ ] I have used Azure Copilot and the SRE Agent with appropriate permissions and approvals.
- [ ] I can position Security Copilot and its E5/E7 inclusion.
- [ ] I have built a Power Automate flow using AI Builder and a Foundry model with private connectivity.

## Maps to

- **AB-100**: agentic AI business solutions architecture (Copilot, Copilot Studio, Agent 365, build vs buy).
- **AI-103**: integrating Foundry agents with Microsoft 365 and Copilot Studio.
- **AZ-305** (adjacent): operations and governance tooling.

## Next

25 · [Security](../25-security/README.md)
