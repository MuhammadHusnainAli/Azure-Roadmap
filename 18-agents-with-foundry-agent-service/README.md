# 18 · Agents with Foundry Agent Service

> Tracks: E ● A ● C ● · Part D · AI on Azure

Foundry Agent Service (formerly Azure AI Agent Service) is Microsoft's managed platform for building, running, publishing and governing AI agents on Azure. This topic starts with what an agent actually is and when not to build one, then covers the platform end to end as it stands in late 2026: prompt agents, voice-based prompt agents and hosted agents; basic, standard and private-network setups; built-in tools, custom tools, MCP, toolboxes, tool search and skills; long-term memory; multi-agent orchestration (including the retirement of Foundry's visual workflows on 1 December 2026 and the move to Agent Framework, Logic Apps and A2A); routines, background mode and resilient long-running agents; voice agents with telephony; the new agent endpoint and publishing model for Microsoft 365 Copilot and Teams; Entra Agent ID and on-behalf-of access; and the design patterns and anti-patterns that make agents reliable.

## What you will be able to do

- Decide whether a use case needs an agent, RAG, a workflow or plain code.
- Choose between prompt, voice-based and hosted agents, and between basic, standard and private-network setups.
- Build prompt agents with versioning, stable endpoints and pinned production versions.
- Package your own framework code as a hosted agent with sessions, state and the right protocol.
- Attach built-in tools and connect your own systems with functions, OpenAPI, Azure Functions, connectors and MCP.
- Curate tools in a toolbox with central auth, tool search and versioning, and package know-how as skills.
- Add long-term memory safely with per-user scope, retention and poisoning defences.
- Orchestrate multi-agent systems with A2A and Agent Framework, with human-in-the-loop steps.
- Trigger agents on schedules and events, run long tasks in the background, and design for crash recovery.
- Build a voice agent and connect it to a phone number.
- Publish agents to Microsoft 365 Copilot and Teams, and control who can call them.
- Give each agent least-privilege access with Entra Agent ID, and use on-behalf-of where user permissions matter.
- Run a design review against Microsoft's patterns and anti-patterns.

## Prerequisites

[Topic 14](../14-microsoft-foundry-platform/README.md) (Foundry resources, projects, roles, SDK), [Topic 15](../15-foundry-models-and-azure-openai/README.md) files 07 and 08 (Responses API, function calling), [Topic 16](../16-prompt-engineering-and-model-customisation/README.md) files 01–04, and [Topic 17](../17-rag-and-knowledge/README.md) files 01, 07 and 08. [Topic 02](../02-identity-and-access/README.md) file 10 (Entra Agent ID) helps for file 13.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Agent Fundamentals](01-agent-fundamentals.md) | 40 min | Python, function tool loop |
| 02 | [Foundry Agent Service Overview](02-foundry-agent-service-overview.md) | 50 min | Portal, Python |
| 03 | [Prompt Agents](03-prompt-agents.md) | 45 min | Versioning, evaluation |
| 04 | [Hosted Agents](04-hosted-agents.md) | 60 min | azd, Agent Framework |
| 05 | [Built-In Tools](05-built-in-tools.md) | 50 min | File search, code interpreter |
| 06 | [Custom Tools](06-custom-tools.md) | 50 min | OpenAPI with managed identity |
| 07 | [MCP Tools, Toolboxes and Skills](07-mcp-tools-toolboxes-and-skills.md) | 55 min | MCP approvals, toolbox, SKILL.md |
| 08 | [Agent Memory](08-agent-memory.md) | 45 min | Memory store, per-user scope |
| 09 | [Multi-Agent Workflows](09-multi-agent-workflows.md) | 55 min | A2A handoff, maker-checker |
| 10 | [Routines and Long-Running Agents](10-routines-and-long-running-agents.md) | 40 min | Routines, background mode |
| 11 | [Voice Agents](11-voice-agents.md) | 45 min | Voice agent, telephony |
| 12 | [Publishing Agents](12-publishing-agents.md) | 40 min | Teams/Copilot publish, A2A endpoint |
| 13 | [Agent Identity and Authorisation](13-agent-identity-and-authorisation.md) | 50 min | RBAC for agent identity, OBO |
| 14 | [Agent Design Patterns and Anti-Patterns](14-agent-design-patterns-and-anti-patterns.md) | 40 min | Refactor a poor design |

**AI engineers:** all of it, with 03–09 and 13 in depth. **Architects:** all of it, with 01, 02, 09, 12, 13 and 14 in depth. **Consultants:** 01, 02, 09, 12 and 14, plus the cost and compliance notes in 05, 07 and 11.

## Completion checklist

- [ ] I can explain when an agent is the wrong answer, using the CAF questions.
- [ ] I can compare prompt, voice-based and hosted agents and the three environment setups.
- [ ] I have built, versioned and pinned a prompt agent, and called its endpoint with a least-privilege identity.
- [ ] I have deployed a hosted agent with azd and right-sized its sandbox.
- [ ] I have used file search, code interpreter and an Azure AI Search tool, and know which tools leave the compliance boundary.
- [ ] I have connected an internal API through OpenAPI with project managed identity.
- [ ] I have connected an MCP server with `allowed_tools` and approvals, and shared tools through a toolbox.
- [ ] I have added per-user memory with a TTL and tested for cross-user leakage and poisoning.
- [ ] I have built an A2A handoff and know the migration options for Foundry workflows before 1 December 2026.
- [ ] I have created a routine and run a background response.
- [ ] I have built a voice agent and tuned turn detection.
- [ ] I have published an agent to Teams and Microsoft 365 Copilot and understand the approval and data-handling implications.
- [ ] I have granted an agent identity least-privilege access and used on-behalf-of for user data.
- [ ] I have reviewed an agent design against the anti-pattern list.

## Maps to

- **AI-103**: build and deploy agents with Foundry Agent Service, tools, knowledge, memory and multi-agent orchestration.
- **AB-100**: design agentic business solutions, publishing to Microsoft 365 and governance with Agent 365.
- **AZ-305**: design secure, governed agent platforms with identity, networking and data residency.
- **SC-500**: secure agent identities, tool access and data flows.

## Next

19 · [Agent Frameworks and Protocols](../19-agent-frameworks-and-protocols/README.md)
