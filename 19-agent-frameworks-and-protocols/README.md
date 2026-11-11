# 19 · Agent Frameworks and Protocols

> Tracks: E ● A ○ C – · Part D · AI on Azure

Topic 18 covered Microsoft's managed agent platform. This topic covers the code and the standards around it: **Microsoft Agent Framework**, the open-source successor to Semantic Kernel and AutoGen, with its agents, harness, middleware and graph-based workflows; how to migrate from Semantic Kernel and the now-archived AutoGen; the **Model Context Protocol** (including the stateless 2026-07-28 revision) and how to build, secure, govern and register MCP servers on Azure Functions, Container Apps, API Management and API Center; the **Agent2Agent protocol** for interoperability between opaque agents; running LangGraph, OpenAI Agents SDK, Claude Agent SDK, CrewAI and LlamaIndex agents on Foundry; the **Microsoft 365 Agents SDK**, Agents Toolkit and Teams SDK for channel delivery; and a selection guide for choosing among all of them.

## What you will be able to do

- Build agents with Microsoft Agent Framework: tools, sessions, middleware, context providers and the harness agent.
- Orchestrate multi-agent workflows (sequential, concurrent, handoff, group chat, Magentic) with checkpoints and human-in-the-loop.
- Plan and execute a migration from Semantic Kernel or AutoGen.
- Explain MCP roles, features, transports, versions and authorisation, and its security risks.
- Build an MCP server, host it on Azure, secure it with Entra ID, govern it with API Management and register it in API Center.
- Explain A2A agent cards, tasks and lifecycle, and connect agents across platforms.
- Host third-party framework agents on Foundry with identity, tools and tracing.
- Deliver agents to Teams, Microsoft 365 Copilot and other channels with the Agents SDK.
- Choose the right framework (or none) for a scenario and standardise the platform around it.

## Prerequisites

[Topic 18](../18-agents-with-foundry-agent-service/README.md) (especially files 01–04, 07, 09 and 13). Python or .NET programming experience. [Topic 11](../11-integration-messaging-and-apis/README.md) files 06–08 (API Management, AI gateway, API Center) help for file 05.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Microsoft Agent Framework](01-microsoft-agent-framework.md) | 50 min | Python agent, tools, middleware |
| 02 | [Agent Framework Workflows](02-agent-framework-workflows.md) | 60 min | Sequential, handoff, checkpoints |
| 03 | [Migrating from Semantic Kernel and AutoGen](03-migrating-from-semantic-kernel-and-autogen.md) | 40 min | Port an SK agent |
| 04 | [Model Context Protocol](04-model-context-protocol.md) | 50 min | FastMCP server, Inspector |
| 05 | [Building and Hosting MCP Servers on Azure](05-building-and-hosting-mcp-servers-on-azure.md) | 60 min | Functions MCP, APIM, API Center |
| 06 | [Agent-to-Agent Protocol](06-agent-to-agent-protocol.md) | 45 min | A2A client and server |
| 07 | [Third-Party Frameworks on Foundry](07-third-party-frameworks-on-foundry.md) | 45 min | Host LangGraph on Foundry |
| 08 | [Microsoft 365 Agents SDK and Toolkit](08-microsoft-365-agents-sdk-and-toolkit.md) | 45 min | Agents SDK + Foundry agent in Teams |
| 09 | [Framework Selection Guide](09-framework-selection-guide.md) | 35 min | Selection workshop |

**AI engineers:** all of it, with 01, 02, 04 and 05 in depth. **Architects:** 01, 04, 05, 06 and 09 in depth; skim 02, 03, 07 and 08. **Consultants:** 09 and the overviews of 01, 04 and 06.

## Completion checklist

- [ ] I have built an Agent Framework agent with a tool, a session and middleware, and run it against a Foundry model.
- [ ] I have built and checkpointed a multi-agent workflow and resumed it after a stop.
- [ ] I can map Semantic Kernel and AutoGen concepts to Agent Framework and explain the support status of each.
- [ ] I can explain MCP hosts, clients, servers, tools, resources, prompts, transports and the main 2026-07-28 changes.
- [ ] I have deployed an MCP server to Azure with Entra authentication, put it behind API Management and registered it in API Center.
- [ ] I can explain A2A agent cards, tasks, states and contextId, and have called a remote agent over A2A.
- [ ] I have hosted a third-party framework agent on Foundry with tracing.
- [ ] I have delivered a Foundry-backed agent to Teams with the Agents SDK or Foundry publishing.
- [ ] I can recommend a framework for a scenario and justify it with clear criteria.

## Maps to

- **AI-103**: implement agents with Microsoft Agent Framework, MCP and multi-agent orchestration.
- **AI-200**: integrate agents and MCP servers into Azure applications and APIs.
- **AB-100**: choose agent build options and interoperability approaches for business solutions.

## Next

20 · [Foundry Tools](../20-foundry-tools/README.md)
