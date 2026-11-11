# Agent-to-Agent Protocol

## Why it matters

MCP connects agents to **tools**. The **Agent2Agent (A2A) protocol** connects agents to **other agents**, possibly built by other teams, on other frameworks, in other clouds. A remote agent is an opaque peer: you don't see its prompts, memory or tools, you just delegate a task and get results, status updates and artifacts back. As organisations accumulate agents (a travel agent in Copilot Studio, a finance agent in Foundry, a vendor's procurement agent elsewhere), A2A is the interoperability layer that lets them collaborate without bespoke integrations. Foundry supports A2A both as a tool (call remote agents) and as an endpoint (be called), with A2A 1.0 GA.

## Key concepts

### MCP vs A2A

| | MCP | A2A |
|---|---|---|
| Connects | Agent ↔ tools, resources, prompts | Agent ↔ agent |
| Remote side | Transparent capabilities with schemas | **Opaque** agent with its own reasoning, memory and tools |
| Interaction | Mostly request/response tool calls | Tasks with lifecycle, multi-turn, streaming, push notifications |
| Typical use | "Look up this order" | "Plan and book this trip", "Investigate this incident" |

They're complementary: an agent can use MCP tools internally and expose itself to others via A2A.

### Core actors

| Actor | Role |
|---|---|
| **User** | Human or service with a goal |
| **A2A client (client agent)** | Acts for the user and initiates A2A requests |
| **A2A server (remote agent)** | Exposes an HTTP endpoint implementing A2A; processes tasks and returns results |

### Communication elements

| Element | Description |
|---|---|
| **Agent Card** | JSON metadata: identity, capabilities, endpoint, **skills**, authentication requirements; discoverable at `https://{domain}/.well-known/agent-card.json` or via registries |
| **Task** | Stateful unit of work with an ID and lifecycle |
| **Message** | One turn (role `user` or `agent`) with a `messageId` and parts |
| **Part** | Exactly one of `text`, `raw` (bytes), `url` (file reference) or `data` (JSON), plus media type and metadata |
| **Artifact** | A deliverable (document, image, structured data) produced during a task; can be streamed |
| **contextId** | Server-generated ID grouping related tasks into one conversation or session |

An agent can respond with a **Message** (immediate answer) or a **Task** (longer work).

### Task lifecycle

```text
submitted ─► working ─► completed
                 │   ├─► failed
                 │   ├─► canceled
                 │   └─► rejected
                 ├─► input-required ─(client sends more input)─► working
                 └─► auth-required  ─(client authenticates)────► working
```

Terminal states (`completed`, `canceled`, `rejected`, `failed`) can't be restarted; follow-ups create **new tasks in the same `contextId`**. Interrupted states (`input-required`, `auth-required`) wait for the client.

### Interaction mechanisms

| Mechanism | Use |
|---|---|
| **Request/response with polling** | Simple calls; poll long tasks |
| **Streaming (SSE)** | Real-time incremental status and artifacts |
| **Push notifications** | Server calls a client webhook for very long tasks or disconnected clients |

### Protocol bindings and versions

A2A defines bindings for **JSON-RPC**, **gRPC** and **HTTP+JSON/REST**. **Version 1.0** is the current stable release; Foundry supports **A2A 1.0 (GA)** and **0.3 (Preview)**. The protocol is an open standard designed for long-running, asynchronous and human-in-the-loop work, with enterprise features such as standard auth schemes, multi-tenancy and extensions.

### Security

- Agent cards declare required authentication (OAuth 2.0/OIDC, API keys, mTLS); use Entra ID on Azure.
- **Trust the publisher**: Microsoft's guidance is to verify you trust both the endpoint and the organisation operating a remote agent before connecting.
- Treat remote agent outputs as untrusted data.
- Decide whether calls carry user context (delegated) or the calling agent's identity (app-only) ([Topic 18 file 13](../18-agents-with-foundry-agent-service/13-agent-identity-and-authorisation.md)).

## How it works on Azure

| Capability | How |
|---|---|
| **Foundry agent calls a remote agent** | `A2ATool(a2a_version=A2AProtocolVersion.V1_0, project_connection_id=...)` on a prompt agent or in a toolbox ([Topic 18 file 09](../18-agents-with-foundry-agent-service/09-multi-agent-workflows.md)) |
| **Foundry agent exposed via A2A** | Enable incoming A2A: publish an agent card and enable the A2A protocol on the agent endpoint (`.../agents/{agent}/endpoint/protocols/a2a`, audience `https://ai.azure.com`) |
| **Hosted agents** | A2A v1.0 endpoint alongside Responses and Invocations |
| **Agent Framework client** | `A2AAgent(url=...)` or `A2AAgent(agent_card=...)` wraps any A2A endpoint as a normal agent (`agent-framework-a2a`) |
| **Agent Framework server** | `agent-framework-a2a` (`A2AExecutor`) or `agent-framework-hosting-a2a` for app-owned executors; .NET ASP.NET Core hosting |
| **Work IQ** | Exposes an A2A interface for delegating tasks to Microsoft 365 intelligence |
| **Copilot Studio** | Can connect to and be called by A2A agents ([Topic 24](../Roadmap.md)) |
| **Agent 365 registry** | Inventory and governance of agents across platforms |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **A2A vs MCP** | MCP for deterministic tools; A2A when you delegate a goal to an autonomous agent owned elsewhere. |
| **A2A vs in-process workflow** | In-process workflows (file 02) for agents you own in one codebase; A2A across team, platform or organisation boundaries. |
| **Discovery** | Well-known URI for public or partner agents; registry (catalogue, Agent 365) inside the enterprise. |
| **Sync vs async** | Streaming for interactive tasks; push notifications for long jobs; design clients for `input-required`. |
| **Version** | Target A2A 1.0; keep 0.3 only for existing integrations. |
| **Identity** | App-only (agent identity) for back-office delegation; delegated user context when the remote agent must honour user permissions. |
| **Contracts** | Publish precise skills in agent cards; version cards; document expected artifacts. |

## Hands-on

1. **Client**: connect to a remote A2A agent with Agent Framework.

```python
# pip install agent-framework-a2a --pre
import asyncio
from agent_framework.a2a import A2AAgent

async def main():
    async with A2AAgent(name="remote", url="https://<a2a-agent-host>") as agent:
        response = await agent.run("Summarise the open incidents for service X.")
        print(response.messages[0].text)

asyncio.run(main())
```

2. **Discovery**: fetch the agent card from `/.well-known/agent-card.json` with `A2ACardResolver` and inspect skills and auth requirements.
3. **Server**: expose an Agent Framework agent with `A2AExecutor` (Starlette/uvicorn), then call it from step 1.
4. **Foundry**: enable incoming A2A on a Foundry prompt agent and call it from a second Foundry agent via `A2ATool`; inspect both traces.
5. Trigger an `input-required` state (ask an ambiguous question) and handle the clarification turn in the same `contextId`.

## Security, cost and reliability notes

- **Opaque remote agents** can do anything their operator allows; contractually and technically limit what data you send.
- **Authentication** on both directions; no anonymous A2A endpoints for enterprise data.
- **Prompt injection across agents**: a compromised remote agent's output can steer your agent; validate and constrain.
- **Cost attribution**: remote agents incur their own costs; agree on metering and limits for cross-team calls.
- **Reliability**: time-outs, retries with idempotency, handling of `failed`/`rejected`, and fallbacks when a remote agent is unavailable. Agent Framework supports stream reconnection (restart from the beginning), not mid-stream resumption.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using A2A for simple tool calls | Use MCP or OpenAPI |
| Ignoring `input-required` and `auth-required` states | Implement interrupted-state handling |
| Restarting completed tasks | Create a new task in the same `contextId` |
| Vague agent cards | Describe skills, inputs and outputs precisely |
| Connecting to unknown public agents | Verify publisher trust and auth first |
| Building new integrations on A2A 0.3 | Use 1.0 |

## Check yourself

1. How does A2A differ from MCP?
2. What is an agent card, and where is it discovered by default?
3. Define task, message, part, artifact and contextId.
4. List the A2A task states, distinguishing terminal and interrupted ones.
5. Name the three interaction mechanisms.
6. Which protocol bindings does A2A define?
7. How do you call a remote A2A agent from Agent Framework?
8. How do you expose a Foundry agent over A2A?

<details>
<summary>Answers</summary>

1. MCP connects agents to transparent tools, resources and prompts; A2A connects agents to opaque peer agents with task lifecycles, multi-turn interaction and artifacts.
2. A JSON document describing identity, endpoint, capabilities, skills and auth requirements; at `https://{domain}/.well-known/agent-card.json` (or from a registry).
3. Task: stateful unit of work; message: one turn with role and parts; part: one content item (text, raw bytes, URL or data); artifact: a deliverable output; contextId: groups related tasks into a conversation.
4. Submitted, working, input-required, auth-required, completed, failed, canceled, rejected; terminal: completed, failed, canceled, rejected; interrupted: input-required, auth-required.
5. Request/response with polling, streaming over SSE, push notifications.
6. JSON-RPC, gRPC and HTTP+JSON/REST.
7. `A2AAgent(url=...)` or `A2AAgent(agent_card=...)` from `agent_framework.a2a`, then `run()`.
8. Enable incoming A2A by publishing an agent card and enabling the A2A protocol on the agent endpoint (REST), exposing `.../endpoint/protocols/a2a`.

</details>

## References

- [A2A protocol specification](https://a2a-protocol.org/latest/specification/)
- [A2A key concepts](https://a2a-protocol.org/latest/topics/key-concepts/)
- [Life of a task](https://a2a-protocol.org/latest/topics/life-of-a-task/)
- [A2A agents in Agent Framework](https://learn.microsoft.com/agent-framework/integrations/by-component/agent-services/a2a)
- [Host Agent Framework agents with A2A](https://learn.microsoft.com/agent-framework/hosting/self-hosting/a2a/)
- [Connect to an A2A endpoint from Foundry](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent)
- [Enable incoming A2A on a Foundry agent](https://learn.microsoft.com/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint)
